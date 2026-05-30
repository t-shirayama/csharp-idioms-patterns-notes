# LINQ を使いこなす

## 概要

`GroupBy`、`Join`、`SelectMany`、`Aggregate`、`OrderBy`、`DistinctBy` などを使うと、集計や変換を短く表現できる。ただし、短く書けることと読みやすいことは同じではない。

LINQ を使いこなすとは、長いチェーンを書けることではなく、データの形が変わる地点を読み手に伝えられること。分類、平坦化、結合、並び替えのどれをしているのかが見出しなしでも追える程度に保つ。

## 使いどころ

`GroupBy` は分類と集計、`SelectMany` は入れ子のコレクションの平坦化、`OrderBy` / `ThenBy` は複数キーの並び替えに向く。C# 10 以降で使える `DistinctBy`、`MinBy`、`MaxBy` は、キーに基づく選択を読みやすくできる。

```csharp
var totalsByCustomer = orders
    .GroupBy(order => order.CustomerId)
    .Select(group => new
    {
        CustomerId = group.Key,
        Total = group.Sum(order => order.Amount)
    })
    .ToList();
```

集計値を複数使う場合は、同じ group を何度も列挙しないように、一度匿名型へまとめると読みやすい。

```csharp
var summaries = orders
    .GroupBy(order => order.CustomerId)
    .Select(group =>
    {
        var items = group.ToList();

        return new CustomerSummary(
            group.Key,
            items.Count,
            items.Sum(order => order.Amount));
    })
    .ToList();
```

`SelectMany` は「親子構造を平坦化して扱う」ときに向く。親の情報も必要なら、匿名型で持ち運ぶ。

```csharp
var shippedLines = orders
    .SelectMany(
        order => order.Lines,
        (order, line) => new { order.OrderId, Line = line })
    .Where(x => x.Line.Status == LineStatus.Shipped)
    .ToList();
```

`DistinctBy`、`MinBy`、`MaxBy` は意図が伝わりやすい。古いコードで `GroupBy(...).Select(g => g.First())` を見かけたら、置き換えられるか検討する。

```csharp
var latestByCustomer = orders
    .OrderByDescending(order => order.CreatedAt)
    .DistinctBy(order => order.CustomerId)
    .ToList();
```

## 判断基準

- 分類して集計するなら `GroupBy`。
- 入れ子の要素を一覧として扱うなら `SelectMany`。
- 片方の集合をキーで引くなら、`Join` と `Dictionary` のどちらが読みやすいか比べる。
- 畳み込みに `Aggregate` を使う前に、`Sum`、`Any`、`All`、`MaxBy` などの専用演算子で表せないか確認する。
- 並び替え条件が複数あるなら `OrderBy` の後に `ThenBy` を使い、2 回目の `OrderBy` で前の順序を捨てない。

## 避けたい書き方

複雑な業務ルールを長い LINQ チェーンに詰め込むと、デバッグとレビューが難しくなる。条件名を変数や小さなメソッドに切り出したほうが、仕様変更にも追従しやすい。

```csharp
// 避けたい例: 条件の意味が追いづらい
var result = orders
    .Where(o => o.Status == OrderStatus.Paid && o.Amount > 0 && !o.IsTest)
    .GroupBy(o => o.CustomerId)
    .Where(g => g.Count() >= 3 && g.Sum(o => o.Amount) > 10000)
    .OrderByDescending(g => g.Max(o => o.CreatedAt))
    .ToList();
```

```csharp
// 良い例: 条件の意味を名前に逃がす
static bool IsBillable(Order order)
{
    return order.Status == OrderStatus.Paid
        && order.Amount > 0
        && !order.IsTest;
}

var result = orders
    .Where(IsBillable)
    .GroupBy(order => order.CustomerId)
    .Select(group => CustomerPurchaseSummary.From(group))
    .Where(summary => summary.IsLoyalCustomer)
    .OrderByDescending(summary => summary.LastPurchasedAt)
    .ToList();
```

`OrderBy` を重ねると、前の並び替えは破棄される。複数キーのソートは `ThenBy` を使う。

```csharp
// 避けたい例: Name の順序だけが残る
var sorted = users
    .OrderBy(user => user.Department)
    .OrderBy(user => user.Name)
    .ToList();
```

```csharp
var sorted = users
    .OrderBy(user => user.Department)
    .ThenBy(user => user.Name)
    .ToList();
```

## テストしやすさ

複雑な LINQ は、途中の形をテストしづらい。業務ルールを `IsBillable` や `CustomerPurchaseSummary.From` のような小さな関数へ切り出すと、LINQ の構造とルールのテストを分けられる。境界値、空コレクション、重複キー、null を含む入力は、LINQ の見た目では見落としやすい。

## レビュー観点

- `GroupBy` 後に同じ group を何度も集計して、不要な列挙を増やしていないか。
- `Join` より `Dictionary` による事前索引化のほうが読みやすい場面ではないか。
- `Aggregate` が単純な `Sum`、`Any`、`All`、`Select` で表現できない理由があるか。
- query syntax と method syntax が混在して、かえって読みづらくなっていないか。
- 2 回目以降のソートに `ThenBy` ではなく `OrderBy` を使っていないか。
- LINQ チェーンが長い場合、途中の業務用語が名前として残っているか。
- `DistinctBy` でどの要素が残るかが、並び順を含めて明確か。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: GroupBy

> 対応元: P2 / `GroupBy`、`DistinctBy`、`ToDictionary` の comparer 指定と代表要素の決まり方を例で補強する。

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
