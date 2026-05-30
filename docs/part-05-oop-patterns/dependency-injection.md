# Dependency Injection

## 概要

Dependency Injection は、クラスが依存オブジェクトを自分で作らず、外から受け取る設計です。C# / .NET では DI コンテナがよく使われますが、本質はコンテナではなく依存方向を整理することです。

## 使いどころ

- 外部 I/O、時刻、乱数、設定、ログなどをテストで差し替えたい。
- アプリケーション層が具体的なインフラ実装に直接依存しないようにしたい。
- ライフタイムをアプリケーション全体で一貫して管理したい。
- 設定、認証、DB、HTTP など、実行環境で実装や値が変わるものを境界として扱いたい。

```csharp
public sealed class OrderService
{
    private readonly IOrderRepository _orders;
    private readonly TimeProvider _timeProvider;

    public OrderService(IOrderRepository orders, TimeProvider timeProvider)
    {
        _orders = orders;
        _timeProvider = timeProvider;
    }

    public Task SubmitAsync(Order order, CancellationToken cancellationToken)
    {
        order.SubmittedAt = _timeProvider.GetUtcNow();
        return _orders.SaveAsync(order, cancellationToken);
    }
}
```

## 判断基準

DI で注入する対象は、単に `new` できるかではなく、呼び出し側から見て差し替える理由があるかで決めます。外部 I/O、現在時刻、ランダム値、設定、認証情報のようにテストや環境で変わるものは注入しやすい候補です。値オブジェクトや計算だけの小さなヘルパーまで interface 化すると、依存関係だけが増えます。

ライフタイムは特に事故が起きやすい領域です。`Singleton` は状態を持たないか、スレッドセーフである必要があります。`Scoped` は Web リクエストやジョブ単位の依存に向き、`Transient` は軽量で状態を共有しないサービスに向きます。

```csharp
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddSingleton(TimeProvider.System);
builder.Services.AddHttpClient<PaymentClient>();
```

## 避けたい書き方

- メソッド内で `new` して外部依存を固定し、テストで差し替えられない。
- Service Locator 的に `IServiceProvider` を渡し、依存関係をコンストラクタから隠す。
- singleton に scoped な依存を持たせるなど、ライフタイムを混ぜる。
- optional な依存を nullable で大量に受け取り、実行時の組み合わせで振る舞いが変わる。

```csharp
public sealed class OrderService
{
    public Task SubmitAsync(Order order)
    {
        var repository = new SqlOrderRepository();
        return repository.SaveAsync(order, CancellationToken.None);
    }
}
```

```csharp
// 避けたい例: 依存が隠れ、テスト時も実行時まで不足に気づきにくい。
public sealed class OrderService(IServiceProvider services)
{
    public Task SubmitAsync(Order order, CancellationToken cancellationToken)
    {
        var repository = services.GetRequiredService<IOrderRepository>();
        return repository.SaveAsync(order, cancellationToken);
    }
}
```

## よくある失敗

- DI コンテナに登録するためだけに interface を作り、抽象化の意味が説明できない。
- コンストラクタ引数が増え続け、クラスが複数のユースケースを抱えているサインを見逃す。
- `IOptions<T>` と生の設定値が混在し、どこで検証されるか分からない。
- テストで mock を細かく設定しすぎ、実装の順序変更だけでテストが壊れる。

## レビュー観点

- コンストラクタを見れば、そのクラスの依存が分かるか。
- DI コンテナの登録と実際のライフタイムが一致しているか。
- options、logger、repository などの依存が過剰に増え、責務が膨らんでいないか。
- テストで差し替えるべき依存と、単なる値オブジェクトを混同していないか。
- `IServiceProvider`、static singleton、環境変数の直接参照で依存が隠れていないか。

---

[Part README に戻る](README.md)


