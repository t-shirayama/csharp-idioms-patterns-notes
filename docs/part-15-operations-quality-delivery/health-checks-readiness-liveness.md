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
