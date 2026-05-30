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

<!-- TODO対応追記 -->

## TODO対応: specification/query object の境界

> 対応元: P2 / specification/query object の境界、`DbContext` を直接使う場合のテスト方針を追加する。

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
