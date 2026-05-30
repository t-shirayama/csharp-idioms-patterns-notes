# ドメインモデルとアプリケーション層の境界

ドメインモデルは、業務ルールと不変条件を表す場所です。アプリケーション層は、ユースケースの手順、トランザクション境界、外部 I/O の呼び出し順を調整する場所です。両者を混ぜると、テストが重くなり、HTTP、DB、メッセージングの都合が業務ルールへ入り込みます。

## 責務の分担

| 要素 | 主な責務 | 置きたい場所 | 避けたい依存 |
| --- | --- | --- | --- |
| Value Object | 値の妥当性、等価性、表現の隠蔽 | Domain | JSON、EF Core、DI container |
| Entity / Aggregate | 識別子、状態遷移、不変条件 | Domain | HTTP request、DbContext |
| Domain Service | Entity だけに置きにくい業務判断 | Domain | Repository 実装、外部 API SDK |
| Application Service / UseCase | 入力 DTO の解釈、トランザクション、Repository 呼び出し | Application | ASP.NET Core の Controller 詳細 |
| Repository interface | 永続化の抽象 | Domain または Application | EF Core の Include 戦略の露出 |
| Infrastructure | Repository 実装、外部 API、ファイル I/O | Infrastructure | 業務判断の分散 |
| DTO | 入出力契約、シリアライズ境界 | Presentation / Application boundary | Entity の公開 |

## 入力 DTO から Entity へ渡す流れ

境界では文字列や数値として受け取り、早い段階で Value Object に変換します。変換できない理由は validation error として返し、Entity には不正な値を渡さない形にします。

```csharp
public sealed record CreateOrderRequest(string CustomerId, string PostalCode);

public sealed record PostalCode
{
    public string Value { get; }

    private PostalCode(string value) => Value = value;

    public static bool TryCreate(string value, out PostalCode? postalCode)
    {
        postalCode = null;
        if (!Regex.IsMatch(value, "^[0-9]{3}-[0-9]{4}$")) return false;

        postalCode = new PostalCode(value);
        return true;
    }
}
```

```csharp
public async Task<Result<OrderDto>> HandleAsync(
    CreateOrderRequest request,
    CancellationToken cancellationToken)
{
    if (!PostalCode.TryCreate(request.PostalCode, out var postalCode))
    {
        return Result.Invalid("郵便番号の形式が不正です。");
    }

    var customer = await customers.GetAsync(request.CustomerId, cancellationToken);
    var order = Order.Create(customer.Id, postalCode!);

    await orders.AddAsync(order, cancellationToken);
    await unitOfWork.SaveChangesAsync(cancellationToken);

    return Result.Success(OrderDto.From(order));
}
```

## 避けたい例

Entity が DTO、HTTP、EF Core の属性、DI container を直接知ると、業務ルールの単体テストまで実行環境に引きずられます。

```csharp
public sealed class Order
{
    public void Apply(CreateOrderRequest request, IServiceProvider services)
    {
        // DTO と DI container に依存しているため、ドメインルールが境界の都合に汚染される。
    }
}
```

## Repository と transaction の境界

Repository interface は「永続化方式を隠す」ために使います。ただし、複雑な検索条件をすべて汎用 Repository に押し込むと、結局 `IQueryable<T>` を漏らしがちです。読み取りは query object や専用 query service、書き込みは aggregate 単位の Repository に分けると、transaction と不変条件を追いやすくなります。

## レビュー観点

- ドメイン層が DTO、EF Core、HTTP、DI container、外部 SDK を参照していないか。
- 入力 DTO の検証後に Value Object を生成し、Entity へ不正値を渡さない構造になっているか。
- Application Service が transaction、外部 I/O、通知、ログの境界を明示しているか。
- 出力 DTO への mapping で内部の Entity をそのまま公開していないか。
- Repository interface が永続化都合を隠しつつ、`IQueryable<T>` の無制限な漏出を避けているか。
