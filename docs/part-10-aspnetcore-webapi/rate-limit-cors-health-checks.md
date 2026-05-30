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
