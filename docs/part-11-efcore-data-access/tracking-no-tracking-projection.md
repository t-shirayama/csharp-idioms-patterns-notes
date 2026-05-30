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

<!-- TODO対応追記 -->

## TODO対応: AsNoTrackingWithIdentityResolution

> 対応元: P1 / `AsNoTrackingWithIdentityResolution`、`Attach` / `Update` の過剰更新、DTO からの部分更新、overposting 対策を追加する。

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
