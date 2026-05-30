# トランザクションと同時実行制御

## 概要

EF Core の `SaveChangesAsync()` は、通常は一回の保存をトランザクション内で実行します。ただし、複数回の保存、複数 aggregate の更新、外部 I/O、同時更新が絡むと、どこまでを一貫して成功させるかを明示する必要があります。

## 判断基準

- 一つの `SaveChangesAsync()` で完結するなら暗黙のトランザクションで足りることが多い
- 複数回の DB 更新を一つの整合性単位にするなら明示的なトランザクションを検討する
- 外部 API、メール送信、キュー投入を DB トランザクション内に入れない
- 同時更新が起きる業務では row version などの楽観的同時実行制御を使う
- リトライしてよい処理と、利用者に競合を返すべき処理を分ける

## 使いどころ

残高更新、在庫引当、予約確定、請求確定のように、複数の値が同時に整合していなければならない処理で重要です。管理画面の単純な名称変更でも、上書き衝突が問題になるなら同時実行制御を検討します。

## 良い例

一つの業務操作を一つの保存にまとめます。

```csharp
var order = await db.Orders
    .SingleAsync(order => order.Id == command.OrderId, cancellationToken);

var stock = await db.Stocks
    .SingleAsync(stock => stock.ProductId == order.ProductId, cancellationToken);

stock.Reserve(order.Quantity);
order.Confirm();

await db.SaveChangesAsync(cancellationToken);
```

複数回の保存が必要な場合は、境界を明示します。

```csharp
await using var transaction =
    await db.Database.BeginTransactionAsync(cancellationToken);

payment.MarkCaptured();
await db.SaveChangesAsync(cancellationToken);

invoice.MarkIssued();
await db.SaveChangesAsync(cancellationToken);

await transaction.CommitAsync(cancellationToken);
```

## 避けたい書き方

DB トランザクション中に外部 I/O を入れると、ロック時間が延び、失敗時の整合性も難しくなります。

```csharp
await using var transaction =
    await db.Database.BeginTransactionAsync(cancellationToken);

order.Confirm();
await mailer.SendConfirmationAsync(order.Email, cancellationToken);

await db.SaveChangesAsync(cancellationToken);
await transaction.CommitAsync(cancellationToken);
```

メール送信が遅い、または失敗した場合、DB ロックや再試行の扱いが複雑になります。

## 改善例

DB 更新と外部通知を分け、必要なら outbox pattern を検討します。

```csharp
order.Confirm();
db.OutboxMessages.Add(OutboxMessage.OrderConfirmed(order.Id));

await db.SaveChangesAsync(cancellationToken);
```

同時更新を検出する場合は、row version を使います。

```csharp
public sealed class Product
{
    public int Id { get; init; }
    public string Name { get; private set; } = "";
    public byte[] RowVersion { get; private set; } = [];
}
```

```csharp
modelBuilder.Entity<Product>()
    .Property(product => product.RowVersion)
    .IsRowVersion();
```

## レビュー観点

- トランザクション境界が業務上の整合性単位と一致しているか
- `SaveChangesAsync()` の回数と失敗時の中途半端な状態を説明できるか
- 外部 I/O が DB トランザクション内に入っていないか
- 同時更新時に上書きでよいのか、競合検出が必要か
- `DbUpdateConcurrencyException` を握りつぶしていないか
- リトライ可能な失敗と、利用者に競合を返す失敗を分けているか
