# generics、variance、constraints

## 概要

generics は、型ごとに重複する処理をまとめながら、コンパイル時の型安全性を保つための仕組みです。`List<T>` のようなコレクションだけでなく、repository、mapper、factory、validation、結果型など、実務の共通部品でよく使います。

重要なのは、`T` を使えば抽象化できるという話ではなく、型パラメータにどの能力を期待するかをコードで表すことです。そのために constraints を使い、必要な場合だけ variance で代入互換性を広げます。

## 判断基準

- 複数の型に対して、同じ意味の処理を型安全に共有したいなら generics を検討する。
- 型ごとに業務ルールが大きく違うなら、generic 化より個別の型や interface を優先する。
- 実装内で `new T()`、比較、非 null、参照型、値型などを前提にするなら constraints を明示する。
- 読み取り専用の戻り値中心なら covariance、入力として受け取る処理中心なら contravariance を検討する。
- 型パラメータが増えすぎる場合は、抽象化の単位を分ける。

## 使いどころ

generics は、結果型、ID型、共通 validation、キャッシュ、変換処理、リポジトリの抽象、テストデータビルダーで有効です。型情報を残したまま共通処理を書けるため、`object` や reflection に比べて呼び出し側のミスを早く検出できます。

constraints は、実装の都合を呼び出し側に伝える契約です。たとえば `where T : notnull` は、辞書キーやキャッシュキーとして null を許さない意図を示せます。

## 良い例

```csharp
public sealed class Cache<TKey, TValue>
    where TKey : notnull
{
    private readonly Dictionary<TKey, TValue> values = [];

    public TValue GetOrAdd(TKey key, Func<TKey, TValue> factory)
    {
        if (values.TryGetValue(key, out var value))
        {
            return value;
        }

        value = factory(key);
        values.Add(key, value);
        return value;
    }
}
```

`TKey` に `notnull` を付けることで、`Dictionary<TKey, TValue>` の前提とAPI契約が一致します。呼び出し側は null を渡せないことを型から読み取れます。

variance は interface や delegate の設計で効きます。

```csharp
public interface IReadOnlyRepository<out T>
{
    T FindById(Guid id);
}

public interface IValidator<in T>
{
    bool IsValid(T value);
}
```

`out T` は戻り値として使う型、`in T` は入力として受け取る型に向きます。mutable な操作を混ぜると成立しないため、責務の分離にもつながります。

## 避けたい書き方

```csharp
public T Create<T>()
{
    return (T)Activator.CreateInstance(typeof(T))!;
}
```

この書き方は、コンパイル時に必要な前提が見えません。引数なしコンストラクタが必要なら `new()` constraint を使うか、factory を受け取る方が安全です。

```csharp
public void Save<T>(T value)
{
    if (value is Order order)
    {
        SaveOrder(order);
    }
    else if (value is Customer customer)
    {
        SaveCustomer(customer);
    }
}
```

型ごとに分岐するだけなら、generic にする意味は薄いです。overload、interface、個別 service に分けた方が意図を追いやすくなります。

## 改善例

```csharp
public interface IEntity
{
    Guid Id { get; }
}

public sealed class InMemoryRepository<TEntity>
    where TEntity : IEntity
{
    private readonly Dictionary<Guid, TEntity> entities = [];

    public void Add(TEntity entity)
    {
        entities.Add(entity.Id, entity);
    }

    public TEntity? Find(Guid id)
    {
        return entities.GetValueOrDefault(id);
    }
}
```

`TEntity` が `Id` を持つことを constraint で表しています。これにより、実装内で reflection や型判定を使わずに済みます。

factory が必要な場合は、`new()` に寄せすぎず、生成責務を外に出す選択もあります。

```csharp
public static TResult Convert<TSource, TResult>(
    TSource source,
    Func<TSource, TResult> converter)
{
    return converter(source);
}
```

生成方法が業務ルールを含むなら、constraint より明示的な delegate の方がテストしやすいです。

## レビュー観点

- generic 化により、呼び出し側の型安全性が上がっているか。
- 型パラメータ名から役割が読み取れるか。
- constraints は実装の前提と一致しているか。
- `object`、reflection、強制キャストを隠すためだけの generic になっていないか。
- variance は読み取り専用、入力専用の責務分離に沿っているか。
- generic API のテストは、代表的な複数の型で確認されているか。
