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

<!-- TODO対応追記 -->

## TODO対応: DbUpdateConcurrencyException の具体的な処理

> 対応元: P0 / `DbUpdateConcurrencyException` の具体的な処理、409 への変換、再読み込み、利用者への競合表示例を追加する。

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
