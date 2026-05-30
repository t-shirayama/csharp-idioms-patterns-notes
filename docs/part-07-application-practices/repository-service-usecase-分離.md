# repository / service / usecase 分離

## 概要

層を分ける目的は、名前の付いた箱を増やすことではなく、変更理由を分けることです。repository は永続化の詳細、application service や usecase はアプリケーションの手順、domain service は特定の Entity に置きにくい業務判断を担当します。

Controller は HTTP の境界に集中させ、認証情報、リクエスト DTO、ステータスコードの変換を扱います。DB クエリ、外部 API 呼び出し、業務判断が Controller に集まると、テストも変更も重くなります。

```csharp
public sealed class SubmitOrderUseCase
{
    private readonly IOrderRepository orderRepository;
    private readonly IPaymentGateway paymentGateway;

    public SubmitOrderUseCase(IOrderRepository orderRepository, IPaymentGateway paymentGateway)
    {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
    }

    public async Task SubmitAsync(Guid orderId, CancellationToken cancellationToken)
    {
        var order = await orderRepository.GetAsync(orderId, cancellationToken)
            ?? throw new OrderNotFoundException(orderId);

        order.Submit();
        await paymentGateway.AuthorizeAsync(order.PaymentId, cancellationToken);
        await orderRepository.SaveAsync(order, cancellationToken);
    }
}
```

## 使いどころ

- Controller を薄くし、ユースケース単位でテストしたい。
- DB、外部 API、現在時刻、認証ユーザーなどの依存を差し替えたい。
- 業務処理の手順を API や UI から再利用したい。

## 責務の置き場所

層の名前よりも、「変更理由」が分かれているかを見ます。

- Controller: HTTP request / response、認証ユーザー、ステータスコード、DTO 変換。
- Usecase / Application Service: 1 つの操作の手順、トランザクション境界、外部依存の呼び出し順。
- Domain Model: 状態遷移、不変条件、業務上の判断。
- Repository: 永続化の詳細、ユースケースに必要な取得・保存。
- Infrastructure adapter: DB、HTTP、メール、メッセージングなど外部 I/O の実装。

```csharp
public sealed class CancelOrderUseCase
{
    private readonly IOrderRepository orders;

    public CancelOrderUseCase(IOrderRepository orders)
    {
        this.orders = orders;
    }

    public async Task<Result<Order>> ExecuteAsync(Guid orderId, CancellationToken cancellationToken)
    {
        var order = await orders.FindAsync(orderId, cancellationToken);
        if (order is null)
        {
            return Result<Order>.Failure("ORDER_NOT_FOUND");
        }

        if (!order.CanCancel())
        {
            return Result<Order>.Failure("ORDER_CANNOT_BE_CANCELLED");
        }

        order.Cancel();
        await orders.SaveAsync(order, cancellationToken);
        return Result<Order>.Success(order);
    }
}
```

この形では、HTTP の都合を知らずにキャンセル手順をテストできます。Controller は Result を HTTP 応答へ変換するだけに寄せます。

## repository の粒度

repository は `Add`、`Update`、`Delete` だけの汎用 CRUD に寄せすぎると、ユースケースの意図が消えます。一方で、すべての LINQ を repository に閉じ込めようとすると、検索条件の組み合わせが爆発します。

```csharp
public interface IOrderRepository
{
    Task<Order?> FindForSubmissionAsync(Guid orderId, CancellationToken cancellationToken);
    Task SaveAsync(Order order, CancellationToken cancellationToken);
}
```

「提出に必要な注文を取得する」のように、ユースケース側の目的を名前に含めると、必要な関連データやロック方針を repository の実装に寄せられます。

## トランザクション境界

usecase はトランザクションの境界を意識します。ただし、外部 API 呼び出しを DB トランザクションの中に入れると、ロック時間が伸び、失敗時の扱いも難しくなります。

```csharp
// よい例: DB の更新と外部通知を分け、通知は後続処理で扱える形にする。
order.Submit();
outbox.Add(OrderSubmitted.Create(order.Id));

await unitOfWork.SaveChangesAsync(cancellationToken);
```

メール送信やメッセージ発行は、outbox pattern などで「DB 更新後に確実に処理する」設計を検討します。小さなアプリでは過剰な場合もありますが、二重送信や更新済みなのに通知できない失敗をどう扱うかは決めておきます。

## 避けたい書き方

- repository に業務フローや HTTP の都合を入れる。
- service という名前の巨大クラスに、検証、永続化、通知、変換をすべて入れる。
- Entity Framework Core の `DbSet` を薄く包むだけの repository を大量に作り、かえって表現力を落とす。
- Controller から `DbContext`、外部 API、domain の状態変更を直接つなぐ。
- domain model が `HttpContext`、`ILogger`、`DbContext` を知っている。

```csharp
// 避けたい例: HTTP 境界、DB、業務判断が Controller に集中している。
public async Task<IActionResult> Submit(Guid id)
{
    var order = await dbContext.Orders.FindAsync(id);
    order.Status = "Submitted";
    await paymentClient.AuthorizeAsync(order.PaymentId);
    await dbContext.SaveChangesAsync();
    return Ok();
}
```

```csharp
// 避けたい例: repository が HTTP 応答を知ってしまっている。
public Task<IActionResult> SaveAndReturnOkAsync(Order order);
```

## レビュー観点

- Controller が HTTP 境界の責務に収まっているか。
- repository が永続化の抽象として意味を持ち、業務フローを抱えていないか。
- usecase / application service の単位が、利用者の操作や業務手順と対応しているか。
- 抽象化がテストや変更に効いているか、それとも単なる転送メソッドになっていないか。
- トランザクション境界と外部 I/O の順序が、失敗時の整合性を考慮しているか。
- domain model がインフラや UI の都合から独立しているか。
- interface が「差し替えたい境界」を表しているか、単なる実装隠しになっていないか。

---

[Part README に戻る](README.md)


