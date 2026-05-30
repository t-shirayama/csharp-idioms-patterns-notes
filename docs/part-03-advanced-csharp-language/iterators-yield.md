# iterators と yield

## 概要

iterator は、`IEnumerable<T>` や `IEnumerator<T>` を返す処理を簡潔に書くための仕組みです。`yield return` を使うと、コレクションを一度に作らず、必要になったタイミングで1件ずつ値を返せます。

`yield` はメモリ効率と読みやすさを改善できますが、遅延評価になります。つまり、メソッドを呼んだ時点では処理が実行されず、列挙されたときに初めて処理が進みます。この評価タイミングを理解しないと、例外、リソース解放、副作用、複数回列挙で事故が起きます。

## 判断基準

- 大きなデータを1件ずつ処理したいなら `yield` を検討する。
- 呼び出し側が一度だけ順に列挙する前提なら `IEnumerable<T>` が自然。
- すぐ全件必要なら `List<T>` や配列を返す方が分かりやすい。
- DB、HTTP、ファイルI/Oなど外部リソースをまたぐ場合は、列挙タイミングと dispose を確認する。
- 非同期に逐次取得するなら `IAsyncEnumerable<T>` も検討する。
- 引数 validation を呼び出し時に失敗させたいなら、iterator 本体と validation を分ける。
- sequence を複数回使う呼び出し側では、`IEnumerable<T>` のまま保持せず、必要なら境界で materialize する。

## 使いどころ

`yield` は、ファイル行の処理、ページング結果の逐次変換、木構造の走査、フィルタリング、テストデータ生成などに向いています。中間リストを作らずに済むため、メモリ使用量を抑えられます。

ただし、業務上「この時点で全件を確定させたい」境界では、あえて `ToList()` で即時評価する方が安全です。

`yield` を含むメソッドは iterator method になり、メソッド本体の実行開始が列挙時まで遅れます。そのため、引数チェックを本体に書くと、呼び出し時ではなく `foreach` や `ToList()` のタイミングで例外が出ます。API として即時に失敗してほしい場合は、外側の通常メソッドと内側の iterator に分けます。

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

引数 validation を即時に行いたい場合は、local function を使います。

```csharp
public static IEnumerable<Category> TraverseSafe(Category root)
{
    ArgumentNullException.ThrowIfNull(root);
    return TraverseCore(root);

    static IEnumerable<Category> TraverseCore(Category node)
    {
        yield return node;

        foreach (var child in node.Children)
        {
            foreach (var item in TraverseCore(child))
            {
                yield return item;
            }
        }
    }
}
```

`TraverseSafe(null)` は呼び出し時に失敗します。`yield` の遅延評価と API の失敗タイミングを分けて考えられます。

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

非同期に逐次取得する場合は、`IAsyncEnumerable<T>` と cancellation を明示します。

```csharp
public static async IAsyncEnumerable<Order> StreamOrdersAsync(
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    await foreach (var order in client.ReadOrdersAsync(cancellationToken))
    {
        yield return order;
    }
}
```

`await foreach` の呼び出し側でもキャンセルを渡せるようにしておくと、長時間の stream を止めやすくなります。

## 使いすぎのサイン

- 呼び出し側が常に `ToList()` しており、遅延評価の利点がない。
- sequence の列挙中に DB 接続やファイルハンドルを長く保持している。
- `yield return` の前後にログ、カウンタ更新、状態変更などの副作用が多い。
- 例外発生タイミングがレビューやテストで説明できない。
- `IEnumerable<T>` を返す public API 名から、一度だけ消費する stream なのか、何度でも列挙できる値の集合なのか分からない。

## レビュー観点

- `IEnumerable<T>` を返すことが、遅延評価として自然か。
- メソッド呼び出し時と列挙時で、例外発生タイミングがずれても問題ないか。
- 複数回列挙で、重い計算、I/O、副作用が繰り返されないか。
- `using`、`try/finally`、外部接続が iterator の寿命と合っているか。
- 即時評価すべき境界で `ToList()` / `ToArray()` が使われているか。
- テストで、途中終了、複数回列挙、空 sequence、例外ケースを確認しているか。

## テスト観点

- iterator method の呼び出し時と列挙時で、どちらのタイミングで例外が出るかを確認する。
- `Take(1)` などで途中終了した場合に、`finally` や `Dispose` が実行されるか。
- 同じ sequence を2回列挙したとき、同じ結果になる設計か、再実行される設計かを明確にする。
- I/O を含む stream は、途中キャンセル、接続失敗、読み取り途中の例外を確認する。
- `IAsyncEnumerable<T>` では、`await foreach` の cancellation と consumer 側の途中終了を確認する。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: IAsyncEnumerable<T> と Task<List<T>> / Chan…

> 対応元: P1 / `IAsyncEnumerable<T>` と `Task<List<T>>` / `Channel<T>` の使い分け、EF Core `IQueryable` との違い、キャンセル伝播の例を追加する。

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
