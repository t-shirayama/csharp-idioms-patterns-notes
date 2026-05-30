# lock / Monitor / thread safety

## 概要

thread safety は「この型を使えば安全」という単純な話ではありません。複数のスレッドやタスクが同じ mutable state を同時に読む、書く、複数操作を組み合わせるときに、途中状態や競合をどう防ぐかの話です。

`lock` は C# で最も基本的な排他手段です。短いクリティカルセクションを同期的に守るには有効ですが、長い処理や外部 I/O を囲むと待ち時間が増え、デッドロックやスループット低下の原因になります。

## 判断基準

- 共有 mutable state を本当に持つ必要があるかを先に確認する。
- 1つの値ではなく、複数の値をまとめて一貫させたいなら排他範囲を明確にする。
- `lock` は短く、同期的で、例外時にも状態が壊れない範囲に限定する。
- `await` を含む処理や外部 I/O の上限制御には `lock` ではなく `SemaphoreSlim` や `Channel` を検討する。

## 使いどころ

メモリ内キャッシュの更新、連番の採番、複数フィールドの一括更新、初期化処理の二重実行防止など、短時間で終わる共有状態の保護に向きます。

逆に、HTTP 呼び出し、DB アクセス、ファイル I/O、長い計算を `lock` で囲むと、待っている側もすべて止まります。処理時間が読めないものは、排他ではなくキュー化や同時実行数制御で設計する方が安全です。

## 良い例

```csharp
public sealed class InMemoryCounter
{
    private readonly object _gate = new();
    private int _value;

    public int Next()
    {
        lock (_gate)
        {
            _value++;
            return _value;
        }
    }
}
```

ロック対象は private な専用オブジェクトにします。`this`、`typeof(...)`、文字列をロックすると、外部コードと意図せず同じロックを共有する危険があります。

## 避けたい書き方

```csharp
private readonly object _gate = new();

public async Task RefreshAsync()
{
    lock (_gate)
    {
        // lock の中で await はできない。
        // 同期的に待つとスレッドを占有し、詰まりやすくなる。
        LoadFromApiAsync().GetAwaiter().GetResult();
    }
}
```

`lock` は同期的な排他です。非同期処理を無理に同期ブロックすると、スレッドプール枯渇やデッドロックの原因になります。

## 改善例

共有状態の更新だけを `lock` に閉じ込め、外部 I/O はロック外で行います。

```csharp
private readonly object _gate = new();
private IReadOnlyList<Product> _cachedProducts = [];

public async Task RefreshAsync(CancellationToken cancellationToken)
{
    var products = await _client.GetProductsAsync(cancellationToken);

    lock (_gate)
    {
        _cachedProducts = products;
    }
}

public IReadOnlyList<Product> GetSnapshot()
{
    lock (_gate)
    {
        return _cachedProducts;
    }
}
```

更新済みの不変に近いスナップショットへ差し替えると、ロック範囲を短くできます。

## レビュー観点

- 共有 mutable state はどれか、ローカル変数や不変データにできないか。
- `lock` 対象は private な専用オブジェクトか。
- ロック範囲は短く、外部 I/O や重い計算を含んでいないか。
- 複数フィールドをまとめて更新する場合、途中状態が外から見えないか。
- `Monitor.Enter` / `Exit` を直接使う必要があるか。通常は `lock` で十分か。
- 非同期処理の排他に `lock` を使おうとしていないか。
