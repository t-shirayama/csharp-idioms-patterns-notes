# primary constructors and modern syntax

## 概要

primary constructor は、型の宣言とコンストラクタ引数を近くに置ける構文です。C# 12 以降では class / struct でも使えます。record の primary constructor と似ていますが、等価性やプロパティ生成の意味は異なります。

このほかにも、target-typed `new`、collection expression、file-local type、required member、raw string literal など、現代 C# には短く書ける構文が増えています。採用時は「短いか」だけでなく、初期化責務、validation、チームの読みやすさ、対象 C# バージョンを確認します。

## 判断基準

- 依存関係を受け取るだけの小さな型なら primary constructor が読みやすい。
- validation、変換、複数の constructor overload が必要なら従来のコンストラクタが分かりやすい。
- record の primary constructor と class の primary constructor の意味を混同しない。
- 新構文は、プロジェクトの `LangVersion`、SDK、チームのスタイルと揃える。
- 「短くなったが意図が消えた」場合は採用しない。

## 使いどころ

primary constructor は、DI で依存を受け取る application service、軽い adapter、validator などで有効です。field への代入だけの constructor を減らせるため、依存関係が型宣言の近くに見えます。

collection expression は、小さな固定データやテストデータで読みやすくなります。raw string literal は JSON、SQL、正規表現など、引用符や改行が多い文字列を扱うときに便利です。

## 良い例

```csharp
public sealed class CreateOrderUseCase(
    IOrderRepository repository,
    OrderValidator validator)
{
    public async Task<Guid> ExecuteAsync(
        CreateOrderCommand command,
        CancellationToken cancellationToken)
    {
        validator.Validate(command);

        var order = Order.Create(command.CustomerId, command.Lines);
        await repository.SaveAsync(order, cancellationToken);
        return order.Id;
    }
}
```

依存を受け取るだけなら、primary constructor で十分読みやすくなります。DI対象の型では、依存関係が先頭で見えるのも利点です。

collection expression は意図が単純なときに使います。

```csharp
private static readonly string[] SupportedCurrencies = ["JPY", "USD", "EUR"];
```

従来の配列初期化より短く、固定値の一覧として読みやすいです。

## 避けたい書き方

```csharp
public sealed class User(string name)
{
    public string Name { get; } = name.Trim();
}
```

一見問題なさそうですが、`name` が null の場合にどこで何が起きるかが見えにくいです。validation が重要な値では、明示的なコンストラクタの方が読みやすい場合があります。

```csharp
public sealed class Service(
    IRepository repository,
    ILogger<Service> logger,
    IClock clock,
    IOptions<AppOptions> options,
    IHttpClientFactory httpClientFactory,
    IValidator<Request> validator)
{
}
```

依存が多すぎる型を primary constructor にしても、設計の問題は解決しません。むしろ責務過多が見えやすくなるだけです。

## 改善例

validation が重要な場合は、従来のコンストラクタで意図を明確にします。

```csharp
public sealed class User
{
    public User(string name)
    {
        if (string.IsNullOrWhiteSpace(name))
        {
            throw new ArgumentException("Name is required.", nameof(name));
        }

        Name = name.Trim();
    }

    public string Name { get; }
}
```

依存が多い場合は、責務を分割します。

```csharp
public sealed class SubmitOrderUseCase(
    OrderLoader loader,
    OrderSubmitter submitter,
    ILogger<SubmitOrderUseCase> logger)
{
    public async Task ExecuteAsync(Guid orderId, CancellationToken cancellationToken)
    {
        var order = await loader.LoadAsync(orderId, cancellationToken);
        await submitter.SubmitAsync(order, cancellationToken);
        logger.LogInformation("Submitted order {OrderId}", orderId);
    }
}
```

依存をただ短く書くのではなく、協調する部品の粒度を見直します。

## レビュー観点

- primary constructor により、依存関係や初期化値が読みやすくなっているか。
- validation、変換、不変条件が必要な型で、短さを優先して意図が薄くなっていないか。
- record と class の primary constructor の違いを理解して使っているか。
- 新構文はプロジェクトの C# バージョン、CI、IDE、analyzer と整合しているか。
- 同じ意味の書き方がファイルごとに混在していないか。
- テスト、serialization、DI、source generator など、周辺ツールへの影響を確認しているか。
