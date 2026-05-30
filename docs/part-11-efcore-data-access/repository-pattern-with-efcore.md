# EF Core と Repository pattern

## 概要

EF Core の `DbContext` と `DbSet<T>` は、すでに Unit of Work と Repository に近い役割を持っています。そのため、すべての Entity に機械的な Repository を作ると、EF Core の機能を薄く包むだけになり、かえってクエリ形状や性能の確認が難しくなることがあります。

## 判断基準

- Repository を作る目的が、テスト、業務語彙、永続化詳細の隠蔽のどれかを明確にする
- 単純な CRUD の薄いラッパーだけなら `DbContext` を直接使う方が読みやすい場合がある
- 複雑な検索条件や aggregate の永続化規則をまとめるなら Repository が役立つ
- `IQueryable<T>` を公開すると、呼び出し側へ EF Core の詳細が漏れやすい
- テストしやすさは Repository だけでなく、UseCase の入力出力や DB provider でも考える

## 使いどころ

ドメイン上の aggregate を取得する、保存前に必ず条件を満たす、複数テーブルを特定の読み方で扱う、といった永続化ルールがある場合に Repository は役立ちます。一方、単純な管理画面や CRUD API では、UseCase から `DbContext` を使い、projection を明示した方が分かりやすいこともあります。

## 良い例

Repository に業務上の取得意図がある例です。

```csharp
public sealed class OrderRepository
{
    private readonly AppDbContext db;

    public OrderRepository(AppDbContext db)
    {
        this.db = db;
    }

    public Task<Order?> FindPendingOrderAsync(
        OrderId orderId,
        CancellationToken cancellationToken)
    {
        return db.Orders
            .Include(order => order.Lines)
            .SingleOrDefaultAsync(
                order => order.Id == orderId && order.Status == OrderStatus.Pending,
                cancellationToken);
    }
}
```

## 避けたい書き方

すべての Entity に同じ CRUD Repository を作ると、抽象化の価値が薄くなります。

```csharp
public interface IRepository<T>
{
    IQueryable<T> Query();
    void Add(T entity);
    void Remove(T entity);
}
```

`IQueryable<T>` をそのまま返すと、呼び出し側が `Include`、tracking、projection、実行タイミングを自由に変更でき、Repository が境界として機能しません。

## 改善例

読み取り用途では、クエリオブジェクトや専用メソッドで結果型まで決める方がレビューしやすいことがあります。

```csharp
public Task<List<OrderSummaryDto>> SearchOrdersAsync(
    OrderSearchCondition condition,
    CancellationToken cancellationToken)
{
    return db.Orders
        .AsNoTracking()
        .Where(order => order.CustomerId == condition.CustomerId)
        .OrderByDescending(order => order.OrderedAt)
        .Select(order => new OrderSummaryDto(
            order.Id,
            order.OrderedAt,
            order.TotalAmount))
        .Take(condition.Limit)
        .ToListAsync(cancellationToken);
}
```

UseCase 側では、業務判断と永続化詳細を混ぜすぎないようにします。

```csharp
var order = await orderRepository.FindPendingOrderAsync(
    command.OrderId,
    cancellationToken);

if (order is null)
{
    return ConfirmOrderResult.NotFoundOrAlreadyConfirmed();
}

order.Confirm();
await db.SaveChangesAsync(cancellationToken);
```

## レビュー観点

- Repository を追加する理由が説明できるか
- `DbContext` の薄い CRUD ラッパーだけになっていないか
- `IQueryable<T>` を公開して責務境界を曖昧にしていないか
- クエリ形状、tracking、projection、実行タイミングをレビューできる形か
- UseCase が業務判断、Repository が永続化の意図を表しているか
- テストのためだけに本番コードの設計を不自然にしていないか
