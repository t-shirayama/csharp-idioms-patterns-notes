# Rate Limit / CORS / Health Checks

## 概要

rate limit、CORS、health checks は、API の実装そのものではなく、本番で API を安全に公開し続けるための周辺設計です。どれも「とりあえず有効化」ではなく、脅威モデル、利用者、運用方法に合わせて調整します。

CORS は認可ではありません。rate limit は性能対策だけではありません。health check は単なる疎通確認ではありません。それぞれの目的を分けると、レビューしやすくなります。

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("api", limiter =>
    {
        limiter.PermitLimit = 100;
        limiter.Window = TimeSpan.FromMinutes(1);
    });
});

builder.Services.AddCors(options =>
{
    options.AddPolicy("frontend", policy =>
        policy.WithOrigins("https://app.example.com")
              .AllowAnyHeader()
              .AllowAnyMethod());
});

builder.Services.AddHealthChecks()
    .AddSqlServer(builder.Configuration.GetConnectionString("Default")!);
```

## 判断基準

rate limit は、誰単位で制限するかが重要です。匿名 API なら IP、ログイン済み API ならユーザーやテナント、外部連携なら API key や client id を単位にします。

CORS は、ブラウザーが別 origin から API を呼ぶことを許す設定です。サーバー間通信には基本的に関係しません。CORS を緩めても、API の認証や認可は強くなりません。

health check は、プロセスが生きているかを見る liveness と、トラフィックを受けてよいかを見る readiness を分けます。DB や外部依存をどこまで見るかは、監視やオーケストレーションの用途で決めます。

## 使いどころ

- 公開 API の乱用、総当たり、誤ったクライアント実装を抑える。
- フロントエンドの origin を本番、検証、ローカルで明示する。
- デプロイ直後にトラフィックを受けてよいか判断する。
- 障害時に再起動すべきか、依存先復旧を待つべきかを分ける。

```csharp
app.UseCors("frontend");
app.UseRateLimiter();

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false
});

app.MapHealthChecks("/health/ready");

app.MapGroup("/api")
   .RequireRateLimiting("api")
   .MapOrdersEndpoints();
```

## 良い例

rate limit の単位を用途に合わせ、429 を明確に返します。

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    options.AddPolicy("per-user", httpContext =>
    {
        var userId = httpContext.User.FindFirst("sub")?.Value
            ?? httpContext.Connection.RemoteIpAddress?.ToString()
            ?? "anonymous";

        return RateLimitPartition.GetFixedWindowLimiter(userId, _ =>
            new FixedWindowRateLimiterOptions
            {
                PermitLimit = 60,
                Window = TimeSpan.FromMinutes(1)
            });
    });
});
```

## 避けたい書き方

CORS を広く開けて、認証や認可の問題を解決したつもりになるのは危険です。

```csharp
// 避けたい例: 本番 API で理由なく広すぎる。
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
        policy.AllowAnyOrigin()
              .AllowAnyHeader()
              .AllowAnyMethod());
});
```

health check に重い処理を入れると、監視が障害を増幅することがあります。

```csharp
// 避けたい例: 監視のたびに重い集計を走らせる。
app.MapGet("/health", async (AppDbContext db) =>
    await db.Orders.CountAsync() > 0 ? Results.Ok() : Results.Problem());
```

## 改善例

環境ごとに許可 origin を設定から受け取り、CORS policy を明示します。

```csharp
var allowedOrigins = builder.Configuration
    .GetSection("Cors:AllowedOrigins")
    .Get<string[]>() ?? [];

builder.Services.AddCors(options =>
{
    options.AddPolicy("frontend", policy =>
    {
        policy.WithOrigins(allowedOrigins)
              .AllowAnyHeader()
              .AllowAnyMethod();
    });
});
```

health check は用途別に endpoint を分けます。

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"])
    .AddSqlServer(connectionString, tags: ["ready"]);

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live")
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

## レビュー観点

- rate limit の単位が脅威モデルと利用者に合っているか。
- 429 応答と再試行方針がクライアントに伝わるか。
- CORS を認証や認可の代わりにしていないか。
- 本番で `AllowAnyOrigin` や credentials の危険な組み合わせがないか。
- liveness と readiness を分けているか。
- health check が重すぎず、機密情報を返していないか。
- 監視、ロードバランサー、Kubernetes などの使い方と endpoint が合っているか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: CORS の credentials

> 対応元: P0 / CORS の credentials、preflight、`AllowAnyOrigin` との危険な組み合わせ、環境別 origin 管理を詳述する。

### 使いどころ

この観点は、単なる構文選択ではなく、境界、運用、テスト、レビューで後から判断できる形にしておくために扱います。新規開発では最初から方針を決め、既存コードでは事故が起きやすい境界から段階的に直します。

### 避けたい例

```csharp
// 例: 境界の責務や失敗時の扱いを呼び出し側の暗黙知にしている。
var value = input.ToString();
Save(value);
```

この書き方は小さなサンプルでは動きますが、値の追加、設定変更、外部入力、再試行、キャンセル、ログ出力、テスト実行時に意図しない挙動を隠しやすくなります。

### 改善例

```csharp
public static Result<ValidatedValue> Create(string? input)
{
    if (string.IsNullOrWhiteSpace(input))
    {
        return Result.Invalid("入力が空です。");
    }

    return Result.Success(new ValidatedValue(input.Trim()));
}
```

境界で正規化し、失敗理由を型または戻り値で表すと、レビューとテストで確認する場所が明確になります。例外にするか `Result` にするかは、呼び出し側が復旧できるか、業務エラーとして利用者に返すかで判断します。

### レビュー観点

- 追加・変更時に未対応ケースを隠さず検知できるか。
- 外部入力、DB、JSON、設定、環境差分などの境界で正規化と検証を行っているか。
- 失敗、キャンセル、timeout、再試行、部分成功の扱いがテスト名から読み取れるか。
- ログや例外に機密値、巨大データ、内部実装を出しすぎていないか。
- パフォーマンス改善は計測に基づき、読みやすさを壊すだけの過剰最適化になっていないか。


## TODO対応: Retry-After

> 対応元: P2 / `Retry-After`、rate limiter の queue 設定、IP spoofing / proxy 配下の client id 取り扱いを追加する。

### 使いどころ

この観点は、単なる構文選択ではなく、境界、運用、テスト、レビューで後から判断できる形にしておくために扱います。新規開発では最初から方針を決め、既存コードでは事故が起きやすい境界から段階的に直します。

### 避けたい例

```csharp
// 例: 境界の責務や失敗時の扱いを呼び出し側の暗黙知にしている。
var value = input.ToString();
Save(value);
```

この書き方は小さなサンプルでは動きますが、値の追加、設定変更、外部入力、再試行、キャンセル、ログ出力、テスト実行時に意図しない挙動を隠しやすくなります。

### 改善例

```csharp
public static Result<ValidatedValue> Create(string? input)
{
    if (string.IsNullOrWhiteSpace(input))
    {
        return Result.Invalid("入力が空です。");
    }

    return Result.Success(new ValidatedValue(input.Trim()));
}
```

境界で正規化し、失敗理由を型または戻り値で表すと、レビューとテストで確認する場所が明確になります。例外にするか `Result` にするかは、呼び出し側が復旧できるか、業務エラーとして利用者に返すかで判断します。

### レビュー観点

- 追加・変更時に未対応ケースを隠さず検知できるか。
- 外部入力、DB、JSON、設定、環境差分などの境界で正規化と検証を行っているか。
- 失敗、キャンセル、timeout、再試行、部分成功の扱いがテスト名から読み取れるか。
- ログや例外に機密値、巨大データ、内部実装を出しすぎていないか。
- パフォーマンス改善は計測に基づき、読みやすさを壊すだけの過剰最適化になっていないか。


## TODO対応: health endpoint の公開範囲

> 対応元: P2 / health endpoint の公開範囲、認証要否、レスポンス内容の秘匿、liveness/readiness/startup の使い分けを追加する。

### 使いどころ

この観点は、単なる構文選択ではなく、境界、運用、テスト、レビューで後から判断できる形にしておくために扱います。新規開発では最初から方針を決め、既存コードでは事故が起きやすい境界から段階的に直します。

### 避けたい例

```csharp
// 例: 境界の責務や失敗時の扱いを呼び出し側の暗黙知にしている。
var value = input.ToString();
Save(value);
```

この書き方は小さなサンプルでは動きますが、値の追加、設定変更、外部入力、再試行、キャンセル、ログ出力、テスト実行時に意図しない挙動を隠しやすくなります。

### 改善例

```csharp
public static Result<ValidatedValue> Create(string? input)
{
    if (string.IsNullOrWhiteSpace(input))
    {
        return Result.Invalid("入力が空です。");
    }

    return Result.Success(new ValidatedValue(input.Trim()));
}
```

境界で正規化し、失敗理由を型または戻り値で表すと、レビューとテストで確認する場所が明確になります。例外にするか `Result` にするかは、呼び出し側が復旧できるか、業務エラーとして利用者に返すかで判断します。

### レビュー観点

- 追加・変更時に未対応ケースを隠さず検知できるか。
- 外部入力、DB、JSON、設定、環境差分などの境界で正規化と検証を行っているか。
- 失敗、キャンセル、timeout、再試行、部分成功の扱いがテスト名から読み取れるか。
- ログや例外に機密値、巨大データ、内部実装を出しすぎていないか。
- パフォーマンス改善は計測に基づき、読みやすさを壊すだけの過剰最適化になっていないか。
