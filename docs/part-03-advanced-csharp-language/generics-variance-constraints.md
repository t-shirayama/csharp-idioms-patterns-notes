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
- `T` のままではレビュー時に意味を追いにくいなら、`TCommand`、`TResult`、`TEntity` のように役割を名前に出す。
- public API では、将来の拡張を理由に constraint を弱くしすぎない。弱い契約は呼び出し側に自由を与える一方で、実装側に unsafe な逃げ道を作りやすい。

## 使いどころ

generics は、結果型、ID型、共通 validation、キャッシュ、変換処理、リポジトリの抽象、テストデータビルダーで有効です。型情報を残したまま共通処理を書けるため、`object` や reflection に比べて呼び出し側のミスを早く検出できます。

constraints は、実装の都合を呼び出し側に伝える契約です。たとえば `where T : notnull` は、辞書キーやキャッシュキーとして null を許さない意図を示せます。

よく使う constraint は、次のように「何を許すか」ではなく「何を前提にしてよいか」で読むと判断しやすくなります。

- `where T : class`: 参照型であることを前提にする。nullable 注釈と併せて null 許容性を読む。
- `where T : struct`: 値型であることを前提にする。`default` が業務的に有効値かは別途確認する。
- `where T : notnull`: null をキーや識別子として扱わない契約を表す。
- `where T : unmanaged`: unmanaged メモリや interop 寄りの処理でだけ検討する。
- `where T : new()`: 引数なし生成を許すが、生成時の validation や依存注入が必要な型には向かない。
- `where T : SomeBase` / `where T : ISomeInterface`: 実装内で使うメンバーを型契約として表す。

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

contravariance は、基底型向けの処理を派生型にも使い回したいときに効きます。

```csharp
public sealed class EntityValidator : IValidator<Entity>
{
    public bool IsValid(Entity value)
    {
        return value.Id != Guid.Empty;
    }
}

IValidator<Order> validator = new EntityValidator();
```

`Order` が `Entity` を継承しているなら、`Entity` を検証できる validator は `Order` も検証できます。入力専用の `in T` にしているから成立する設計です。

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

variance を強制キャストで回避するのも避けます。

```csharp
var orders = new List<Order>();
var entities = (IList<Entity>)orders;
```

`IList<T>` は追加も削除もできる mutable collection なので、covariance にはできません。`IReadOnlyList<Entity>` や `IEnumerable<Entity>` として公開できるか、API の責務を見直します。

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

variance が必要な場合は、読み取りと書き込みを分けると API が安定します。

```csharp
public interface IEventPublisher<in TEvent>
{
    Task PublishAsync(TEvent value, CancellationToken cancellationToken);
}

public interface IEventStream<out TEvent>
{
    IEnumerable<TEvent> ReadAll();
}
```

同じ interface に `Publish` と `ReadAll` を混ぜると `in` / `out` を付けられません。variance は、型互換性だけでなく責務分離の設計サインとして使えます。

## 使いすぎのサイン

- 型パラメータが3つ以上あり、呼び出し側が型推論に頼らないと読めない。
- `where` 句が長く、実際には特定の基底型や interface に寄せた方が分かりやすい。
- generic method の中で型判定、reflection、`dynamic` が増えている。
- テストが「1種類の型で通る」だけで、抽象化した意味を確認できていない。
- エラーメッセージが型パラメータ名だけになり、業務上の失敗理由が読み取りにくい。

## レビュー観点

- generic 化により、呼び出し側の型安全性が上がっているか。
- 型パラメータ名から役割が読み取れるか。
- constraints は実装の前提と一致しているか。
- `object`、reflection、強制キャストを隠すためだけの generic になっていないか。
- variance は読み取り専用、入力専用の責務分離に沿っているか。
- generic API のテストは、代表的な複数の型で確認されているか。

## テスト観点

- 代表的な参照型、値型、派生型で同じ契約が守られるか。
- constraint 違反がコンパイル時に検出される設計になっているか。
- `notnull` を使う API では、null を受け取った場合の挙動を nullable 警告と実行時 guard のどちらで守るか。
- variance を使う interface / delegate では、基底型と派生型の代入互換性が期待どおりか。
- factory delegate を受け取る API では、失敗、例外、cancellation が呼び出し側に自然に伝播するか。

---

[Part README に戻る](README.md)
