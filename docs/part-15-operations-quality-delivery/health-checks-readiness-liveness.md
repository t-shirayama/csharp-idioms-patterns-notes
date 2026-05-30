# health checks、readiness、liveness

## 概要

Health check は「アプリケーションが動いているか」を見るだけの endpoint ではありません。liveness、readiness、startup を分けないと、依存先障害時に不要な再起動を起こしたり、まだ初期化中のアプリへリクエストを流したりします。

liveness はプロセスを再起動すべきかを判断するための確認です。readiness はリクエストを受けてよいかを判断するための確認です。startup は起動時の重い初期化や warm-up が終わったかを見るために使います。

## 判断基準

- liveness は軽く、依存先を叩かない。
- readiness は DB、cache、message broker など、リクエスト処理に必須な依存先を必要に応じて見る。
- startup は初期化完了まで readiness と分けて扱う。
- health response は詳細すぎる内部情報や secret を返さない。
- 監視用、ロードバランサー用、Kubernetes probe 用で目的を分ける。

## 使いどころ

ASP.NET Core の Web API、worker service、Kubernetes 上のアプリ、ロードバランサー配下のアプリで使います。特に rolling deployment、DB migration、外部 API 障害、キュー滞留があるシステムでは、readiness の設計がリリースの安定性に直結します。

## 良い例

tag で liveness と readiness を分け、endpoint ごとに実行する check を変えます。

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: new[] { "live" })
    .AddSqlServer(
        builder.Configuration.GetConnectionString("Default")!,
        name: "database",
        tags: new[] { "ready" });

var app = builder.Build();

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live")
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

## 避けたい書き方

liveness で DB や外部 API を確認すると、依存先障害のたびにアプリが再起動される可能性があります。

```csharp
app.MapHealthChecks("/health");
```

すべての check を同じ endpoint にまとめると、監視、ロードバランサー、再起動制御が同じ判断になってしまいます。

## 改善例

目的ごとに endpoint を分け、外部依存は readiness に寄せます。

```csharp
app.MapGet("/health/live", () => Results.Ok(new { status = "alive" }));

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";
        await context.Response.WriteAsJsonAsync(new
        {
            status = report.Status.ToString()
        });
    }
});
```

詳細なエラー内容は監視基盤やログに残し、公開 endpoint には最小限の状態だけを返します。

## レビュー観点

- liveness が外部依存や重い処理を含んでいないか。
- readiness が、実際にリクエスト処理に必要な依存関係を確認しているか。
- DB migration 中、cache 障害、外部 API 障害で期待どおりに traffic を止められるか。
- health endpoint が認証なしで公開される場合、内部情報を返しすぎていないか。
- timeout が設定され、health check 自体が詰まらないか。
- Kubernetes probe、load balancer、監視アラートの目的に合う endpoint が分かれているか。

<!-- TODO対応追記 -->

## TODO対応: timeout

> 対応元: P1 / timeout、degraded、依存先別 readiness、Kubernetes probe 設定例、migration/warm-up 中の挙動を追加する。

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
