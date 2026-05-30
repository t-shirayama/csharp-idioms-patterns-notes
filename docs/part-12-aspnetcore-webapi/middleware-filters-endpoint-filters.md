# Middleware / Filters / Endpoint Filters

## 概要

ASP.NET Core には横断処理を置く場所が複数あります。middleware は HTTP pipeline 全体、MVC filter は Controller / Action、endpoint filter は Minimal API の endpoint 境界に効きます。

どこにでも書ける処理ほど、置き場所の判断が大切です。例外変換、相関 ID、ログ、認可補助、入力の前処理などを handler に散らすと、API が増えるほど一貫性が崩れます。

```csharp
app.UseExceptionHandler();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();

app.MapGroup("/api")
   .AddEndpointFilter<CorrelationEndpointFilter>()
   .MapOrdersEndpoints();
```

## 判断基準

middleware は、リクエスト全体に必ず通したい処理に向きます。例外処理、相関 ID、リクエストログ、認証、認可、圧縮、CORS などです。

MVC filter は Controller / Action の文脈が必要な処理に向きます。model validation の扱い、action 実行前後の処理、結果の整形などです。

endpoint filter は Minimal API の endpoint 単位や route group 単位で、handler の前後に軽い横断処理を入れる用途に向きます。業務判断を深く入れる場所ではありません。

## 使いどころ

- 例外を全 API で同じ形に変換する。
- request id や correlation id をログ scope に追加する。
- route group 単位で入力前処理や簡単な validation をかける。
- Controller と Minimal API の境界で、横断処理の実装方法をそろえる。

```csharp
public sealed class CorrelationEndpointFilter(ILogger<CorrelationEndpointFilter> logger)
    : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var traceId = context.HttpContext.TraceIdentifier;

        using (logger.BeginScope(new Dictionary<string, object?>
        {
            ["TraceId"] = traceId
        }))
        {
            return await next(context);
        }
    }
}
```

## 良い例

横断的な例外処理は handler に書かず、pipeline の境界に集めます。

```csharp
builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<ApiExceptionHandler>();

app.UseExceptionHandler();
```

```csharp
public sealed class ApiExceptionHandler(
    IProblemDetailsService problemDetailsService,
    ILogger<ApiExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        logger.LogError(exception, "Unhandled API exception");

        httpContext.Response.StatusCode = StatusCodes.Status500InternalServerError;

        return await problemDetailsService.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = httpContext,
            Exception = exception,
            ProblemDetails =
            {
                Title = "Unexpected error",
                Status = StatusCodes.Status500InternalServerError
            }
        });
    }
}
```

## 避けたい書き方

各 endpoint で同じ try/catch やログ scope を書くと、漏れや差分が増えます。

```csharp
// 避けたい例: endpoint ごとに例外応答の形が変わりやすい。
app.MapGet("/customers/{id}", async (Guid id, CustomerService service) =>
{
    try
    {
        return Results.Ok(await service.GetAsync(id));
    }
    catch (Exception ex)
    {
        return Results.Json(new { error = ex.Message }, statusCode: 500);
    }
});
```

また、middleware に業務処理を入れると、ルーティングやテストの見通しが悪くなります。

## 改善例

endpoint は成功系の読みやすさを保ち、失敗の変換は共通境界に寄せます。

```csharp
app.MapGet("/customers/{id}", async (
        Guid id,
        CustomerQueries queries,
        CancellationToken cancellationToken) =>
    {
        var customer = await queries.FindAsync(id, cancellationToken);
        return customer is null ? Results.NotFound() : Results.Ok(customer);
    })
   .RequireAuthorization("Customers.Read");
```

共通化する対象は「どの API でも同じ判断」だけにします。対象リソースの所有者確認のような業務判断は use case 側に残します。

## レビュー観点

- middleware、filter、endpoint filter の責務が説明できるか。
- pipeline の順序が認証、認可、例外処理、CORS の要件に合っているか。
- handler や controller に同じ横断処理が重複していないか。
- filter に業務ロジックを入れすぎて、単体テストしにくくなっていないか。
- 短絡した場合でもログ、メトリクス、エラー応答が一貫しているか。
- request body や response body を読む処理が性能やストリーム状態を壊していないか。

---

[Part README に戻る](README.md)
