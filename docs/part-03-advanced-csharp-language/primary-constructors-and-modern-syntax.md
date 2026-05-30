# primary constructors と現代構文

## 概要

primary constructor は、型の宣言とコンストラクタ引数を近くに置ける構文です。C# 12 以降では class / struct でも使えます。record の primary constructor と似ていますが、等価性やプロパティ生成の意味は異なります。

このほかにも、target-typed `new`、collection expression、file-local type、required member、raw string literal など、現代 C# には短く書ける構文が増えています。採用時は「短いか」だけでなく、初期化責務、validation、チームの読みやすさ、対象 C# バージョンを確認します。

## 判断基準

- 依存関係を受け取るだけの小さな型なら primary constructor が読みやすい。
- validation、変換、複数の constructor overload が必要なら従来のコンストラクタが分かりやすい。
- record の primary constructor と class の primary constructor の意味を混同しない。
- 新構文は、プロジェクトの `LangVersion`、SDK、チームのスタイルと揃える。
- 「短くなったが意図が消えた」場合は採用しない。
- primary constructor parameter を property として公開したい場合は、明示的な property 初期化を書く。class では record のように public property が自動生成されない。
- `required` は object initializer を使う呼び出し側への契約であり、実行時 validation の代わりにはならない。

## 使いどころ

primary constructor は、DI で依存を受け取る application service、軽い adapter、validator などで有効です。field への代入だけの constructor を減らせるため、依存関係が型宣言の近くに見えます。

collection expression は、小さな固定データやテストデータで読みやすくなります。raw string literal は JSON、SQL、正規表現など、引用符や改行が多い文字列を扱うときに便利です。

新構文は、既存コードのノイズを減らすときに効果があります。一方で、初期化順序、null 契約、serialization、DI container、source generator など、周辺ツールが構文の意味をどう解釈するかも確認します。言語機能だけで判断せず、プロジェクト全体の読みやすさに合わせます。

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

class の primary constructor parameter を公開 API にしたい場合は、property を明示します。

```csharp
public sealed class Money(decimal amount, string currency)
{
    public decimal Amount { get; } = amount;
    public string Currency { get; } = currency;
}
```

record と違い、class では `Amount` や `Currency` が自動では公開されません。値オブジェクトとして等価性も欲しいなら、record の方が自然な場合があります。

raw string literal は、テスト用 JSON のように構造をそのまま読みたい場合に向きます。

```csharp
var json = """
{
  "status": "submitted",
  "amount": 1200
}
""";
```

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

`required` を validation の代わりにするのも避けます。

```csharp
public sealed class CreateUserRequest
{
    public required string Email { get; init; }
}
```

`required` は初期化時に値を指定させる仕組みであり、空文字やメール形式までは保証しません。入力検証は validator や constructor guard と組み合わせます。

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

required member は、DTO や options の初期化漏れを減らすために使います。

```csharp
public sealed class MailOptions
{
    public required string FromAddress { get; init; }
    public int RetryCount { get; init; } = 3;
}
```

ただし、設定ファイルの bind 結果や serializer が `required` をどう扱うかはライブラリによって差があります。起動時検証や options validation で補います。

target-typed `new` は、左辺から型が明確なときに留めます。

```csharp
private readonly Dictionary<Guid, Order> orders = new();
```

右辺だけを見て型が分からない場所や、複数 overload が絡む場所では、明示的な型を書いた方がレビューしやすいです。

## 使いすぎのサイン

- 新構文への置き換えが目的になり、validation や例外メッセージが薄くなっている。
- primary constructor の引数が多く、型の責務過多を隠している。
- record、class、struct の primary constructor の意味が混ざり、等価性や公開 property の期待がずれている。
- `required`、nullable、constructor guard の責務分担が曖昧になっている。
- collection expression や target-typed `new` が増えすぎて、型を推測するために前後の文脈を読み続ける必要がある。

## C# 13/14 の扱い

新しい構文は、言語バージョンが上がるたびに一律で導入するものではありません。Microsoft Learn の language versioning では、既定の C# バージョンは TFM と連動します。たとえば .NET 9 は C# 13、.NET 10 は C# 14 が既定になります。SDK だけ新しくても、プロジェクトの TFM、CI、IDE、analyzer が揃っていなければ、チーム全体の標準としてはまだ早いことがあります。

C# 13 の `params` collections や新しい `lock` 対応は、ライブラリや低レベルな共通部品では有効ですが、通常の業務ロジックでは「既存の書き方より意図が明確になるか」を先に見ます。C# 14 の `Span<T>` / `ReadOnlySpan<T>` まわりの強化や extension member 系の構文も、性能やAPI設計の理由がある場所から慎重に採用します。

採用判断は、次の3段階で考えるとレビューしやすくなります。

- アプリ全体で使う候補: primary constructor、raw string literal、collection expression のように読みやすさへ直接効くもの。
- ライブラリや高性能処理で使う候補: `Span<T>`、`ref struct`、generic math、`params` collections のように制約や性能理由が強いもの。
- 保留する候補: preview 機能、チームの IDE / formatter / analyzer が追いついていない構文、説明より短さが勝ってしまう構文。

## レビュー観点

- primary constructor により、依存関係や初期化値が読みやすくなっているか。
- validation、変換、不変条件が必要な型で、短さを優先して意図が薄くなっていないか。
- record と class の primary constructor の違いを理解して使っているか。
- 新構文はプロジェクトの C# バージョン、CI、IDE、analyzer と整合しているか。
- 同じ意味の書き方がファイルごとに混在していないか。
- テスト、serialization、DI、source generator など、周辺ツールへの影響を確認しているか。

## テスト観点

- primary constructor に置き換えても、DI container が期待どおり解決できるか。
- validation を従来 constructor に残す場合、null、空文字、範囲外値の例外が変わっていないか。
- record と class の置き換えでは、等価性、serialization、model binding への影響を確認する。
- `required` member は、object initializer、serializer、configuration binding の各入口で初期化漏れを検出できるか。
- collection expression や target-typed `new` への置き換えで、公開 API の型や overload 解決が変わっていないか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: C# 13/14 の具体例 / params collections

> 対応元: P0 / C# 13/14 の具体例を追加する。 / `params` collections、`System.Threading.Lock`、C# 14 extension members、null-conditional assignment、`field` backed properties を「採用してよい場面 / 保留すべき場面」で整理する。

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
