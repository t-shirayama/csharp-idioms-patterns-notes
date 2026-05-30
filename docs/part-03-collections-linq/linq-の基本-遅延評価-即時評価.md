# LINQ の基本、遅延評価、即時評価

## 概要

LINQ は `Where`、`Select`、`Any`、`First` などを組み合わせて、コレクション操作を宣言的に書ける。多くの演算子は遅延評価され、`ToList`、`ToArray`、`Count`、`First` などのタイミングで実際に列挙される。

実務では「何を抽出するか」だけでなく、「いつ評価されるか」「何回評価されるか」が重要になる。特に `IEnumerable<T>` を変数に入れた時点では、まだ何も処理されていないことが多い。

## 使いどころ

条件抽出や射影は LINQ に向いている。途中結果を明確に固定したい、複数回使いたい、DB クエリをここで実行したい、といった場面では `ToList` や `ToArray` で即時評価する。

```csharp
var activeNames = users
    .Where(user => user.IsActive)
    .Select(user => user.DisplayName)
    .ToList();
```

`First` と `Single` は意味が違う。先頭の1件でよいなら `First`、1件だけ存在するというルールを検証したいなら `Single` を使う。

```csharp
var primaryAddress = addresses.FirstOrDefault(address => address.IsPrimary);

if (primaryAddress is null)
{
    return Address.Empty;
}
```

```csharp
// 「1件だけ存在する」は業務ルールとして検証する
var contract = contracts.Single(contract => contract.IsActive);
```

### 遅延評価を意識する

`Where` や `Select` は列挙されるたびに実行される。元のコレクションが変わると、同じ query でも結果が変わる。

```csharp
var activeUsers = users.Where(user => user.IsActive);

users.Add(new User("u-003", isActive: true));

var count = activeUsers.Count(); // 追加後の状態で評価される
```

途中結果を「その時点の結果」として扱いたいなら、意図的に materialize する。

```csharp
var activeUsers = users
    .Where(user => user.IsActive)
    .ToList();
```

### 即時評価する演算子

`ToList`、`ToArray`、`Count`、`Any`、`First`、`Single`、`Sum` などは、その場で列挙を開始する。`OrderBy` や `GroupBy` は遅延評価の形を取るが、列挙時には内部で要素を保持するため、メモリ使用量にも注意する。

## 避けたい書き方

LINQ の中で外部状態を変更すると、いつ何回実行されるかが読みづらくなる。遅延評価と副作用の組み合わせは、バグの再現性も悪くしやすい。

```csharp
// 避けたい例: 列挙されるまで副作用が起きない
var query = users.Select(user =>
{
    auditLog.Add(user.Id);
    return user.DisplayName;
});
```

```csharp
// 良い例: 副作用は列挙処理として明示する
foreach (var user in users)
{
    auditLog.Add(user.Id);
}

var names = users
    .Select(user => user.DisplayName)
    .ToList();
```

`FirstOrDefault` を値型に使うと、見つからなかったのか既定値なのか分かりにくくなる。`SingleOrDefault` も「0件または1件」を許すだけで、複数件なら例外になる。

```csharp
// 避けたい例: 0 が実データなのか未検出なのか曖昧
var amount = orders
    .Where(order => order.Id == id)
    .Select(order => order.Amount)
    .FirstOrDefault();
```

```csharp
var order = orders.FirstOrDefault(order => order.Id == id);

if (order is null)
{
    return NotFound();
}
```

## 判断基準

- query を変数に置くときは、遅延評価のままでよいかを確認する。
- 複数回使う、ログに出す、デバッグで中身を固定したいなら `ToList()` を検討する。
- 1件の取得では `First`、`Single`、`FirstOrDefault`、`SingleOrDefault` の意味を選び分ける。
- `Any` は存在確認、`Count` は件数そのものが必要なときに使う。
- LINQ の中に副作用を入れず、変更処理は `foreach` に分ける。

## テストしやすさ

遅延評価の query は、テストで列挙するまで例外が出ない。フィルタ条件のバグを早く検出したいなら、テストでは `ToList()` で評価して結果を確認する。現在時刻、乱数、外部状態を LINQ の中で参照すると、列挙タイミングで結果が変わるため、テストが不安定になりやすい。

## レビュー観点

- `FirstOrDefault` の戻り値が null になり得ることを扱っているか。
- `Single` を使うことで、データ不整合を検知したい意図があるか。
- `ToList` の位置が早すぎて、不要な中間コレクションを作っていないか。
- 遅延評価される query を複数回列挙しても問題ないか。
- `Select` の中でログ出力、状態変更、例外変換などの副作用をしていないか。
- DB LINQ の場合、C# 上で自然に見える式が SQL として期待通りに評価されるか。

---

[Part README に戻る](README.md)


