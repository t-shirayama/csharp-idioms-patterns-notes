# tracking / no tracking / projection

## 概要

EF Core のクエリでは、Entity を tracking するか、`AsNoTracking()` で読み取り専用にするか、DTO へ projection するかを最初に考えます。これは単なる性能オプションではなく、「このデータを更新するのか」「画面や API に必要な形だけでよいのか」を表す設計判断です。

## 判断基準

- 更新する Entity は tracking して取得する
- 読み取り専用の一覧や詳細表示は `AsNoTracking()` を検討する
- API レスポンスや画面表示は Entity ではなく DTO へ projection する
- 大量データでは tracking によるメモリ使用量と変更検出コストを確認する
- 更新処理で DTO から Entity を再構成する場合は、意図しない全項目更新に注意する

## 使いどころ

一覧 API、検索画面、レポート、CSV 出力のように DB の値を読むだけの処理では、`AsNoTracking()` と projection が基本候補になります。逆に、取得した Entity に対して業務ルールを適用し、変更を保存する処理では tracking が自然です。

## 良い例

読み取り専用の一覧では、必要な列だけを DTO に投影します。

```csharp
var orders = await db.Orders
    .AsNoTracking()
    .Where(order => order.CustomerId == customerId)
    .OrderByDescending(order => order.OrderedAt)
    .Select(order => new OrderSummaryDto(
        order.Id,
        order.OrderedAt,
        order.TotalAmount))
    .Take(50)
    .ToListAsync(cancellationToken);
```

更新する場合は、対象 Entity を取得して業務ルールを通してから保存します。

```csharp
var order = await db.Orders
    .SingleAsync(order => order.Id == command.OrderId, cancellationToken);

order.ChangeShippingAddress(command.Address);

await db.SaveChangesAsync(cancellationToken);
```

## 避けたい書き方

一覧表示で Entity を丸ごと取得すると、不要な列、不要な tracking、意図しないナビゲーション参照が混ざりやすくなります。

```csharp
var orders = await db.Orders
    .Where(order => order.CustomerId == customerId)
    .ToListAsync();

return orders.Select(order => new OrderSummaryDto(
    order.Id,
    order.OrderedAt,
    order.TotalAmount));
```

この例では projection が DB ではなくメモリ上で行われます。件数が増えると読み込み量と tracking コストが大きくなります。

## 改善例

projection を `ToListAsync()` より前に置き、DB 側で必要な列だけを取得します。

```csharp
var orders = await db.Orders
    .AsNoTracking()
    .Where(order => order.CustomerId == customerId)
    .Select(order => new OrderSummaryDto(
        order.Id,
        order.OrderedAt,
        order.TotalAmount))
    .ToListAsync(cancellationToken);
```

更新で DTO をそのまま attach する場合は、更新範囲を明示します。

```csharp
var order = await db.Orders
    .SingleAsync(order => order.Id == command.OrderId, cancellationToken);

order.Rename(command.DisplayName);

await db.SaveChangesAsync(cancellationToken);
```

## レビュー観点

- 読み取り専用クエリで tracking している理由があるか
- API や画面に Entity を直接返していないか
- `Select` が `ToListAsync()` より前にあり、DB 側で projection されるか
- 更新処理で変更してよい列だけが変更される設計か
- 大量データで tracking によるメモリ使用量が問題にならないか
- `AsNoTracking()` を付けた Entity を後で更新しようとしていないか
