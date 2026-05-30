# Concurrent collections

## 概要

`ConcurrentDictionary<TKey,TValue>`、`ConcurrentQueue<T>`、`ConcurrentBag<T>` などの concurrent collection は、複数スレッドからの単体操作を安全にするためのコレクションです。

ただし、collection 自体が thread-safe でも、業務上の一連の判断が自動で安全になるわけではありません。特に「存在確認してから追加する」「取得して更新して保存する」のような複数操作には注意が必要です。

## 判断基準

- 複数スレッドから同じ collection を読み書きする必要があるか。
- 単体操作で十分か、複数操作をまとめた不変条件があるか。
- 順序が必要か。必要なら `ConcurrentQueue<T>` や `Channel` を検討する。
- producer / consumer の完了通知や backpressure が必要なら `Channel` の方が向いている。

## 使いどころ

メモリ内キャッシュ、重複排除、処理済みIDの記録、複数ワーカーからの結果収集などに使えます。単純な共有 `Dictionary` を `lock` で囲むより、意図が伝わりやすい場面があります。

ただし、集計や状態遷移のように複数の値を同時に一貫させたい場合は、collection だけでなく排他や設計分離が必要です。

## 良い例

```csharp
private readonly ConcurrentDictionary<string, CustomerSummary> _cache = new();

public CustomerSummary GetOrCreate(string customerId)
{
    return _cache.GetOrAdd(customerId, id => LoadSummary(id));
}
```

`GetOrAdd` を使うと、`ContainsKey` の後に `Add` する競合を避けられます。

## 避けたい書き方

```csharp
if (!_cache.ContainsKey(customerId))
{
    _cache[customerId] = LoadSummary(customerId);
}

return _cache[customerId];
```

この書き方は check-then-act です。存在確認から追加までの間に、別の処理が同じキーを追加する可能性があります。

## 改善例

値の作成が重い場合や副作用がある場合は、factory が複数回呼ばれても問題ないようにするか、`Lazy<T>` を組み合わせます。

```csharp
private readonly ConcurrentDictionary<string, Lazy<CustomerSummary>> _cache = new();

public CustomerSummary GetOrCreate(string customerId)
{
    var lazy = _cache.GetOrAdd(
        customerId,
        id => new Lazy<CustomerSummary>(() => LoadSummary(id)));

    return lazy.Value;
}
```

`ConcurrentDictionary.GetOrAdd` の value factory は、競合時に複数回呼ばれる可能性があります。外部 API 呼び出しやDB更新のような副作用を直接入れる場合は、その副作用が重複してもよいかを確認します。

## レビュー観点

- 通常の `Dictionary` を複数スレッドから触っていないか。
- `ContainsKey`、`TryGetValue`、追加、更新を別々に呼び、競合条件を作っていないか。
- `GetOrAdd` や `AddOrUpdate` の factory に副作用や重い処理がないか。
- collection の thread safety と、業務上の不変条件を混同していないか。
- 列挙順や完全なスナップショット性に依存していないか。
- キュー処理なら、完了通知や backpressure が必要ではないか。
