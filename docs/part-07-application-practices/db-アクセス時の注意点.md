# DB アクセス時の注意点

## 概要

DB アクセスでは、接続、トランザクション、クエリ数、遅延読み込み、migration、同時更新を意識します。ORM を使っていても、発行される SQL、更新範囲、失敗時の状態は消えません。

特に N+1、意図しない lazy loading、長すぎるトランザクションは、実装時には目立たず本番データで問題化しやすい領域です。repository の抽象化も、DB の特性を完全に隠すのではなく、ユースケースに必要な問い合わせを表す形にすると扱いやすくなります。

```csharp
// 必要な関連データを明示して、ループ内の追加クエリを避ける。
var orders = await dbContext.Orders
    .Include(order => order.Items)
    .Where(order => order.CustomerId == customerId)
    .ToListAsync(cancellationToken);
```

## 使いどころ

- 複数テーブルを更新し、途中失敗時に一貫性を守りたい。
- 一覧画面や集計処理で、クエリ数と取得列を制御したい。
- migration を通じてスキーマ変更をレビュー可能にしたい。
- 同時更新が起きる業務で、楽観ロックや再試行方針を決めたい。

## クエリ設計

DB アクセスでは、取得する行数、列数、関連データ、追跡の有無を明示します。一覧や参照だけの画面では、Entity 全体を読み込むより DTO へ projection する方が軽く、公開項目も分かりやすくなります。

```csharp
var orders = await dbContext.Orders
    .AsNoTracking()
    .Where(order => order.CustomerId == customerId)
    .Select(order => new OrderListItem(
        order.Id,
        order.Status,
        order.OrderedAt,
        order.TotalAmount))
    .ToListAsync(cancellationToken);
```

`Include` は便利ですが、一覧に必要ない巨大な関連データまで読み込むと遅くなります。更新する aggregate を取得するのか、画面表示用に読むのかで、クエリの形を分けます。

## トランザクションと外部 I/O

DB トランザクションは短く保ちます。トランザクション中に HTTP 呼び出し、メール送信、ファイルアップロードなどを行うと、ロック時間が伸び、外部依存の遅延が DB の詰まりに伝播します。

```csharp
await using var transaction = await dbContext.Database
    .BeginTransactionAsync(cancellationToken);

order.MarkAsPaid();
await dbContext.SaveChangesAsync(cancellationToken);

await transaction.CommitAsync(cancellationToken);
```

外部通知が必要な場合は、DB 更新と同じトランザクションで outbox に記録し、別プロセスや後続処理で送信する設計を検討します。すべてのシステムで必要ではありませんが、二重送信や送信漏れの影響が大きい場合は重要です。

## 同時更新

同じ行を複数ユーザーや複数 worker が更新する可能性があるなら、競合を検出する方法を決めます。EF Core では row version などの concurrency token を使えます。

```csharp
public sealed class OrderEntity
{
    public Guid Id { get; set; }
    public OrderStatus Status { get; set; }
    public byte[] RowVersion { get; set; } = Array.Empty<byte>();
}
```

競合時に自動リトライするのか、利用者へ再読み込みを促すのかは業務次第です。金額、在庫、承認状態のように競合の影響が大きい項目は、最後に保存した人が勝つ設計にしない方が安全です。

## migration の見方

スキーマ変更はコード変更と同じくレビュー対象です。カラム追加だけでなく、既存データ、インデックス、ロック時間、ロールバック、段階リリースを確認します。

- NOT NULL カラム追加時に既存行へ値を入れる方針があるか。
- 大きなテーブルへの index 追加でロックや時間が問題にならないか。
- カラム削除の前にアプリ側が参照しなくなっているか。
- migration とアプリのデプロイ順序に互換性があるか。

## 避けたい書き方

- ループの中で DB を毎回問い合わせ、N+1 を作る。
- lazy loading に依存し、どこで SQL が発行されるか読みづらくする。
- トランザクションを広く取りすぎて、外部 API 呼び出しまで含める。
- repository を汎用 CRUD だけにして、ユースケース固有の問い合わせ意図を表せなくする。
- 参照だけのクエリで tracking を有効にしたまま大量データを読む。
- `ToListAsync()` を早く呼びすぎて、以降の絞り込みをメモリ上で行う。

```csharp
// 避けたい例: 注文ごとに Items の問い合わせが発生しやすい。
foreach (var order in orders)
{
    total += order.Items.Sum(item => item.Price);
}
```

```csharp
// 避けたい例: DB で絞れる条件をメモリ上で処理している。
var orders = await dbContext.Orders.ToListAsync(cancellationToken);
return orders.Where(order => order.CustomerId == customerId);
```

## レビュー観点

- 一覧やバッチで N+1 が起きないクエリになっているか。
- トランザクション境界が短く、外部 I/O を含めていないか。
- migration にデータ移行、ロールバック、既存データへの影響が考慮されているか。
- 同時更新時の競合検出、再試行、利用者への通知方針があるか。
- repository の抽象が DB の現実を隠しすぎず、必要な問い合わせを明確にしているか。
- projection、`AsNoTracking()`、pagination、index の必要性を検討しているか。
- `CancellationToken` が DB 呼び出しまで渡っているか。
- 本番データ量でのクエリ数、実行計画、タイムアウトを確認する手段があるか。

---

[Part README に戻る](README.md)


