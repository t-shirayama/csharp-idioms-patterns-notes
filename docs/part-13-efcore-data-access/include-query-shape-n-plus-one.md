# Include / クエリ形状 / N+1

## 概要

`Include` は「関連データも取る」ための便利指定ですが、実際には SQL の join、結果セットの重複、読み込む行数、メモリ使用量を変えます。レビューでは C# の見た目だけでなく、発行される SQL とデータ量を確認します。

## 判断基準

- レスポンスや処理に必要な関連データだけを読み込む
- 一覧では Entity graph より DTO projection を優先する
- 複数 collection の `Include` は cartesian explosion に注意する
- ループ内でナビゲーション参照による追加クエリが出ないか確認する
- 必要に応じて `AsSplitQuery()`、projection、明示的な集計を検討する

## 使いどころ

詳細画面のように、一つの aggregate と関連データをまとめて扱う場合は `Include` が分かりやすいことがあります。一覧、検索、API レスポンスでは、必要な列と件数が明確になりやすい projection が向いています。

## 良い例

一覧では必要な形に直接 projection します。

```csharp
var customers = await db.Customers
    .AsNoTracking()
    .OrderBy(customer => customer.Name)
    .Select(customer => new CustomerListItemDto(
        customer.Id,
        customer.Name,
        customer.Orders.Count))
    .Take(100)
    .ToListAsync(cancellationToken);
```

詳細で collection を複数読む場合は、split query を検討します。

```csharp
var customer = await db.Customers
    .AsNoTracking()
    .Include(customer => customer.Orders)
    .Include(customer => customer.Contacts)
    .AsSplitQuery()
    .SingleAsync(customer => customer.Id == customerId, cancellationToken);
```

## 避けたい書き方

N+1 は、ループ内のナビゲーション参照や追加クエリで発生します。

```csharp
var customers = await db.Customers.ToListAsync(cancellationToken);

foreach (var customer in customers)
{
    Console.WriteLine(customer.Orders.Count);
}
```

lazy loading が有効な場合、このコードは顧客ごとに注文を読む可能性があります。小さい開発データでは気づきにくい問題です。

## 改善例

必要な集計をクエリ内に含めます。

```csharp
var summaries = await db.Customers
    .AsNoTracking()
    .Select(customer => new CustomerOrderSummary(
        customer.Id,
        customer.Name,
        customer.Orders.Count))
    .ToListAsync(cancellationToken);
```

関連データが本当に必要な詳細取得では、対象を一件に絞ってから読み込みます。

```csharp
var order = await db.Orders
    .AsNoTracking()
    .Include(order => order.Lines)
    .SingleAsync(order => order.Id == orderId, cancellationToken);
```

## レビュー観点

- `Include` で読み込む関連データが実際に使われているか
- 一覧 API で Entity graph を返そうとしていないか
- ループ内で DB クエリが増えないか
- 複数 collection の `Include` で結果セットが膨らまないか
- SQL ログや実行計画でクエリ形状を確認できるか
- projection、`AsSplitQuery()`、ページングの検討がされているか
