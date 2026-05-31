# IEnumerable / ICollection / IReadOnlyCollection / IReadOnlyList の使い分け

## 概要

コレクションを API の境界でどう表すかは、呼び出し側に何を約束するかの設計でもある。`IEnumerable<T>` は列挙できることだけを表し、`ICollection<T>` は件数や追加削除を含み、`IReadOnlyCollection<T>` は読み取り専用の件数付きコレクションを表す。順序とインデックスアクセスまで約束するなら、`IReadOnlyList<T>` も候補になる。

「何でも受けられるように `IEnumerable<T>`」は便利だが、内部で `Count()` や複数回列挙をするなら、実際にはもっと強い契約を要求している。引数と戻り値の型は、実装都合ではなく、呼び出し側が誤解しない単位で選ぶ。

## 使いどころ

公開する戻り値は、必要以上に具体型を見せないほうが変更に強い。ただし、呼び出し側が件数を必ず使うなら `IReadOnlyCollection<T>`、インデックスアクセスが必要なら `IReadOnlyList<T>` のように、利用実態に合わせて選ぶ。

```csharp
public IReadOnlyCollection<Order> GetPendingOrders()
{
    return _orders
        .Where(order => order.Status == OrderStatus.Pending)
        .ToList();
}
```

### 引数で選ぶ

- 1 回だけ列挙すればよいなら `IEnumerable<T>`。
- 件数が必要なら `IReadOnlyCollection<T>`。
- 順序とインデックスアクセスが必要なら `IReadOnlyList<T>`。
- メソッド内で追加や削除をする責務があるなら `ICollection<T>` や具体型を検討する。

```csharp
public decimal CalculateTotal(IEnumerable<OrderLine> lines)
{
    return lines.Sum(line => line.UnitPrice * line.Quantity);
}
```

```csharp
public bool HasEnoughItems(IReadOnlyCollection<Item> items)
{
    return items.Count >= 10;
}
```

### 戻り値で選ぶ

戻り値は「呼び出し側に何をしてよいと言うか」を表す。内部では `List<T>` で作っても、外に出すときは変更可能性を狭める。

```csharp
public IReadOnlyList<string> GetWarnings()
{
    return _warnings.ToArray();
}
```

`IReadOnlyCollection<T>` はコレクション構造の変更を防ぐ契約であり、要素オブジェクトの中身まで不変にするものではない。必要なら immutable な DTO や record を使う。

## 避けたい書き方

API 境界で遅延評価の `IEnumerable<T>` をそのまま返すと、呼び出し側の列挙タイミングで DB 接続、外部状態、副作用が絡むことがある。特に `DbContext` やファイル読み込みの寿命を越えて列挙される設計は避けたい。

```csharp
// 避けたい例: DbContext の寿命を越えて評価される可能性がある
public IEnumerable<Order> GetPendingOrders()
{
    return _db.Orders.Where(order => order.Status == OrderStatus.Pending);
}
```

```csharp
// 良い例: 境界の内側で実行し、結果を固定する
public async Task<IReadOnlyList<Order>> GetPendingOrdersAsync(
    CancellationToken cancellationToken)
{
    return await _db.Orders
        .Where(order => order.Status == OrderStatus.Pending)
        .ToListAsync(cancellationToken);
}
```

`IEnumerable<T>` を受け取ってから複数回列挙する実装も注意する。呼び出し側が generator、DB クエリ、ファイル読み込みを渡した場合、同じ処理が何度も走る。

```csharp
// 避けたい例: lines が複数回列挙される
public decimal AverageAmount(IEnumerable<OrderLine> lines)
{
    return lines.Sum(line => line.Amount) / lines.Count();
}
```

```csharp
public decimal AverageAmount(IReadOnlyCollection<OrderLine> lines)
{
    return lines.Count == 0
        ? 0
        : lines.Sum(line => line.Amount) / lines.Count;
}
```

## 判断基準

- 「列挙できる」以上の前提を持つなら、引数型に反映する。
- API 境界で遅延評価を返すときは、列挙タイミングを仕様として説明できる形にする。
- 戻り値を `IEnumerable<T>` にする場合でも、内部で `ToList()` してから返すほうが安全な場面がある。
- `IReadOnlyCollection<T>` は呼び出し側に `Count` を安く使わせたいときに有効。
- 順序が意味を持つなら `IReadOnlyCollection<T>` ではなく `IReadOnlyList<T>` を選ぶ。

## テストしやすさ

引数が必要以上に具体型だと、テストで無関係な準備が増える。一方で、`IEnumerable<T>` のまま複数回列挙するコードは generator を渡すテストで初めて壊れることがある。境界の型と列挙回数は、テストで意図的に確認しておく。

## レビュー観点

- メソッド引数は本当に `IEnumerable<T>` で足りるか、件数や複数回列挙が必要なら明示しているか。
- 戻り値で変更可能な `List<T>` をそのまま公開していないか。
- `IReadOnlyCollection<T>` が「読み取り専用ビュー」であって、要素自体の不変性までは保証しないことを理解しているか。
- 遅延評価を API 境界に持ち込んだとき、列挙タイミングがレビューで追えるか。
- `ICollection<T>` を引数にしている場合、本当に呼び出し先で追加や削除をする責務があるか。
- DB、ファイル、HTTP など寿命のあるリソースをまたぐ `IEnumerable<T>` を返していないか。

---

[Part README に戻る](README.md)


