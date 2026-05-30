# LINQ で避けたい書き方とパフォーマンス

## 概要

LINQ の性能問題は、たいてい「多重列挙」「不要な中間コレクション」「大量データでのメモリ使用」「LINQ to Objects と DB LINQ の混同」から起きる。最初から過剰に最適化する必要はないが、データ量と評価タイミングは意識しておきたい。

レビューでは、LINQ そのものを避けるのではなく、「読みやすさに見合うコストか」「件数が増えたときに計算量が跳ねないか」を見る。数十件の管理画面と、数百万件のバッチでは同じ書き方でも判断が変わる。

## 使いどころ

小から中規模のメモリ上コレクションでは、読みやすい LINQ を優先してよい。大量データ、ホットパス、DB クエリ、ストリーミング処理では、列挙回数や SQL 変換結果を確認しながら使う。

```csharp
// 件数そのものが不要なら Count() より Any()
if (orders.Any(order => order.Status == OrderStatus.Pending))
{
    NotifyPendingOrders();
}
```

`Any`、`All`、`First` は条件を満たした時点で止まる。`Where(...).ToList().Any()` のように、途中で全件を確定させる必要がないなら避ける。

```csharp
// 避けたい例: 全件リスト化してから存在確認している
var hasError = logs
    .Where(log => log.Level == LogLevel.Error)
    .ToList()
    .Any();
```

```csharp
var hasError = logs.Any(log => log.Level == LogLevel.Error);
```

検索を繰り返す場合は、事前に索引を作る。`foreach` の内側に `FirstOrDefault` や `Any` があると、O(n * m) になりやすい。

```csharp
var productsBySku = products.ToDictionary(
    product => product.Sku,
    StringComparer.OrdinalIgnoreCase);

foreach (var line in lines)
{
    if (productsBySku.TryGetValue(line.Sku, out var product))
    {
        ApplyProduct(line, product);
    }
}
```

## 避けたい書き方

`IEnumerable<T>` を複数回列挙すると、そのたびに計算や I/O が走る可能性がある。複数回使うなら、意図的に materialize してから扱う。

```csharp
var expensiveQuery = LoadOrders().Where(order => order.Amount > 0);

// 避けたい例: LoadOrders() が複数回評価される可能性がある
var count = expensiveQuery.Count();
var total = expensiveQuery.Sum(order => order.Amount);

// 必要なら一度固定する
var orders = expensiveQuery.ToList();
var fixedCount = orders.Count;
var fixedTotal = orders.Sum(order => order.Amount);
```

DB クエリに変換される LINQ では、C# のメソッド呼び出しが SQL に変換できない場合がある。`AsEnumerable` 以降はメモリ上処理になるため、データ量が大きい場面では特に注意する。

```csharp
// 避けたい例: 全件取得後にメモリ上で絞り込む可能性がある
var users = db.Users
    .AsEnumerable()
    .Where(user => IsTargetUser(user))
    .ToList();
```

DB 側で絞れる条件は DB LINQ の中に残す。C# の独自ロジックが必要なら、取得件数を先に絞れる条件がないかを確認する。

```csharp
var candidates = await db.Users
    .Where(user => user.IsActive)
    .Where(user => user.CreatedAt >= threshold)
    .ToListAsync(cancellationToken);

var users = candidates
    .Where(IsTargetUser)
    .ToList();
```

## 判断基準

- 存在確認は `Any`、件数が必要なときだけ `Count` を使う。
- `foreach` の中で別コレクションを検索していたら、`Dictionary` や `HashSet` を検討する。
- `ToList` は「ここで固定したい」「複数回使う」「DB クエリをここで実行する」という理由がある場所に置く。
- `OrderBy`、`GroupBy`、`ToList` は全体を保持しやすいので、大量データではメモリを意識する。
- 性能改善を入れるなら、想定件数、呼び出し頻度、計測結果のどれかを説明できるようにする。

## 落とし穴

- `Count()` は `ICollection<T>` なら安いが、ただの `IEnumerable<T>` では全列挙になることがある。
- `Distinct`、`GroupBy`、`ToDictionary` は等価性比較に依存する。文字列では comparer を明示する。
- `Enumerable` の LINQ と `Queryable` の LINQ は別物。`IQueryable<T>` にメソッドを足すと SQL 変換の可否が問題になる。
- `AsEnumerable` は以降の処理をメモリ上に切り替える合図。便利だが、データ量の境界を曖昧にしやすい。

## テストしやすさ

性能問題は単体テストだけでは見つかりにくい。少量データの正常系に加えて、重複キー、空コレクション、数千件程度の入力で処理時間や SQL の形を確認すると、`foreach` 内検索や N+1 を早めに見つけやすい。

## レビュー観点

- `Count() > 0` で存在確認していないか。`Any()` で足りる場面ではないか。
- `Where(...).ToList().Select(...)` のように、途中で不要に materialize していないか。
- LINQ チェーンが長くなりすぎて、途中の意図やデータ量が見えなくなっていないか。
- EF Core などの DB LINQ で、発行される SQL や N+1 問題を確認すべき処理ではないか。
- 性能改善が必要な根拠を、推測ではなく計測やデータ量の見積もりで説明できるか。
- 内側ループで `Any`、`FirstOrDefault`、`SingleOrDefault` を繰り返していないか。
- `AsEnumerable` や `ToList` によって、DB 側で絞れる処理をメモリ上へ移していないか。

---

[Part README に戻る](README.md)


