# Minimal API / Controller の境界

## 概要

ASP.NET Core では、Minimal API と Controller のどちらでも Web API を作れます。選択は好みだけではなく、API の複雑さ、チームの慣れ、OpenAPI 契約、filter、versioning、テスト方針で判断します。

Minimal API は小さく明示的な endpoint に向きます。Controller は属性、filter、model binding、規約、まとまった action 群を扱いやすいです。どちらを選んでも、handler や action に業務ロジックを詰め込まないことが大切です。

```csharp
app.MapGroup("/orders")
   .WithTags("Orders")
   .RequireAuthorization("Orders.Read")
   .MapOrdersEndpoints();
```

## 判断基準

Minimal API は、endpoint 数が少ない、ルートごとの構成をコードで読みたい、軽量なサービスや BFF を作る、route group でまとまりを作れる場合に向きます。

Controller は、action が多い、既存の MVC filter や conventions を使う、versioning や複雑な model binding が多い、チームが Controller ベースの設計に慣れている場合に向きます。

どちらの場合も、API 層の責務は HTTP との変換です。業務判断は use case や application service に置き、永続化詳細は repository や DbContext の境界に閉じ込めます。

## 使いどころ

- 小規模な API や BFF で Minimal API を使う。
- 複雑な業務 API や既存 MVC 資産がある領域で Controller を使う。
- route group や controller 単位で認可、タグ、versioning をまとめる。
- endpoint から use case を呼び、HTTP と業務結果の変換だけを行う。

```csharp
public static class OrderEndpoints
{
    public static RouteGroupBuilder MapOrdersEndpoints(this RouteGroupBuilder group)
    {
        group.MapGet("/{id:guid}", GetOrderAsync)
             .Produces<OrderResponse>()
             .ProducesProblem(StatusCodes.Status404NotFound);

        return group;
    }

    private static async Task<IResult> GetOrderAsync(
        Guid id,
        GetOrderUseCase useCase,
        CancellationToken cancellationToken)
    {
        var result = await useCase.GetAsync(id, cancellationToken);
        return result is null ? Results.NotFound() : Results.Ok(result);
    }
}
```

## 良い例

Minimal API でも handler を小さく保ち、route group に共通設定を寄せます。

```csharp
var orders = app.MapGroup("/api/orders")
    .WithTags("Orders")
    .RequireAuthorization("Orders.Read")
    .RequireRateLimiting("per-user");

orders.MapGet("/{id:guid}", async (
        Guid id,
        GetOrderUseCase useCase,
        CancellationToken cancellationToken) =>
    {
        var order = await useCase.GetAsync(id, cancellationToken);
        return order is null ? Results.NotFound() : Results.Ok(order);
    });
```

Controller の場合も、action は HTTP 変換に集中させます。

```csharp
[ApiController]
[Route("api/orders")]
[Authorize(Policy = "Orders.Read")]
public sealed class OrdersController(GetOrderUseCase useCase) : ControllerBase
{
    [HttpGet("{id:guid}")]
    public async Task<ActionResult<OrderResponse>> Get(
        Guid id,
        CancellationToken cancellationToken)
    {
        var order = await useCase.GetAsync(id, cancellationToken);
        return order is null ? NotFound() : Ok(order);
    }
}
```

## 避けたい書き方

Minimal API の lambda にすべてを詰め込むと、短く始めたはずのコードが最も読みにくい場所になります。

```csharp
// 避けたい例: HTTP、validation、業務判断、DB 更新が一箇所に混ざっている。
app.MapPost("/orders", async (CreateOrderRequest request, AppDbContext db) =>
{
    if (string.IsNullOrWhiteSpace(request.ProductCode))
    {
        return Results.BadRequest();
    }

    var product = await db.Products.SingleAsync(x => x.Code == request.ProductCode);
    var order = new Order(product.Id, request.Quantity);
    db.Orders.Add(order);
    await db.SaveChangesAsync();

    return Results.Ok(order);
});
```

Controller でも、action が use case、mapping、永続化、例外変換を抱えると同じ問題が起きます。

## 改善例

入力 DTO、use case、結果変換を分け、API 層の責務を薄くします。

```csharp
app.MapPost("/orders", async (
        CreateOrderRequest request,
        CreateOrderUseCase useCase,
        CancellationToken cancellationToken) =>
    {
        var command = new CreateOrderCommand(
            request.ProductCode,
            request.Quantity);

        var result = await useCase.CreateAsync(command, cancellationToken);

        return result switch
        {
            CreateOrderResult.Success success =>
                Results.Created($"/orders/{success.OrderId}", success),
            CreateOrderResult.ProductNotFound =>
                Results.Problem(statusCode: 404, title: "Product not found"),
            CreateOrderResult.InvalidQuantity =>
                Results.ValidationProblem(new Dictionary<string, string[]>
                {
                    ["quantity"] = ["Quantity must be greater than zero."]
                }),
            _ => Results.Problem(statusCode: 500)
        };
    })
   .RequireAuthorization("Orders.Write");
```

## レビュー観点

- Minimal API と Controller の選択理由が規模や複雑さに合っているか。
- handler や action が HTTP 変換に集中しているか。
- request DTO、command、domain model、response DTO が混ざっていないか。
- route group や controller 単位で認可、rate limit、tags、versioning がまとまっているか。
- OpenAPI に出る status code と response type が実装と一致しているか。
- use case を単体テストでき、endpoint は統合テストで契約確認できるか。

---

[Part README に戻る](README.md)
