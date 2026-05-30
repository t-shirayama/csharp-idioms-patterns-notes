# iterators、yield

## 概要

iterator は、`IEnumerable<T>` や `IEnumerator<T>` を返す処理を簡潔に書くための仕組みです。`yield return` を使うと、コレクションを一度に作らず、必要になったタイミングで1件ずつ値を返せます。

`yield` はメモリ効率と読みやすさを改善できますが、遅延評価になります。つまり、メソッドを呼んだ時点では処理が実行されず、列挙されたときに初めて処理が進みます。この評価タイミングを理解しないと、例外、リソース解放、副作用、複数回列挙で事故が起きます。

## 判断基準

- 大きなデータを1件ずつ処理したいなら `yield` を検討する。
- 呼び出し側が一度だけ順に列挙する前提なら `IEnumerable<T>` が自然。
- すぐ全件必要なら `List<T>` や配列を返す方が分かりやすい。
- DB、HTTP、ファイルI/Oなど外部リソースをまたぐ場合は、列挙タイミングと dispose を確認する。
- 非同期に逐次取得するなら `IAsyncEnumerable<T>` も検討する。

## 使いどころ

`yield` は、ファイル行の処理、ページング結果の逐次変換、木構造の走査、フィルタリング、テストデータ生成などに向いています。中間リストを作らずに済むため、メモリ使用量を抑えられます。

ただし、業務上「この時点で全件を確定させたい」境界では、あえて `ToList()` で即時評価する方が安全です。

## 良い例

```csharp
public static IEnumerable<string> ReadNonEmptyLines(TextReader reader)
{
    string? line;
    while ((line = reader.ReadLine()) is not null)
    {
        if (!string.IsNullOrWhiteSpace(line))
        {
            yield return line;
        }
    }
}
```

1行ずつ読みながら、空行だけを除外します。巨大な入力でも全行をメモリに載せる必要がありません。

木構造の走査にも向いています。

```csharp
public static IEnumerable<Category> Traverse(Category root)
{
    yield return root;

    foreach (var child in root.Children)
    {
        foreach (var item in Traverse(child))
        {
            yield return item;
        }
    }
}
```

呼び出し側は必要なところまで列挙できます。

## 避けたい書き方

```csharp
public static IEnumerable<Order> LoadOrders()
{
    using var connection = OpenConnection();

    foreach (var order in connection.Query<Order>("select * from orders"))
    {
        yield return order;
    }
}
```

`using` と `yield` の組み合わせ自体はコンパイラが iterator の dispose に変換しますが、呼び出し側が列挙を長時間保持すると接続も長く保持されます。外部リソースをまたぐ場合は、呼び出し側の利用パターンまで含めて判断します。

```csharp
var orders = FindOrders();
var count = orders.Count();
var total = orders.Sum(x => x.Amount);
```

`FindOrders()` が遅延評価なら、2回列挙されます。DBやファイルに接続する sequence では性能問題や結果の不整合につながります。

## 改善例

境界で全件確定させる場合は、即時評価を明示します。

```csharp
var orders = FindOrders().ToList();
var count = orders.Count;
var total = orders.Sum(x => x.Amount);
```

リソースを短く閉じたいなら、読み取り処理内で materialize する選択もあります。

```csharp
public static IReadOnlyList<Order> LoadOrders()
{
    using var connection = OpenConnection();
    return connection.Query<Order>("select * from orders").ToList();
}
```

大量データを本当に逐次処理したいなら、API名で意図を出します。

```csharp
public static IEnumerable<Order> StreamOrders()
{
    using var connection = OpenConnection();

    foreach (var order in connection.Query<Order>("select * from orders"))
    {
        yield return order;
    }
}
```

`Stream` という名前にすることで、呼び出し側に列挙中リソースを保持する可能性を意識させます。

## レビュー観点

- `IEnumerable<T>` を返すことが、遅延評価として自然か。
- メソッド呼び出し時と列挙時で、例外発生タイミングがずれても問題ないか。
- 複数回列挙で、重い計算、I/O、副作用が繰り返されないか。
- `using`、`try/finally`、外部接続が iterator の寿命と合っているか。
- 即時評価すべき境界で `ToList()` / `ToArray()` が使われているか。
- テストで、途中終了、複数回列挙、空 sequence、例外ケースを確認しているか。
