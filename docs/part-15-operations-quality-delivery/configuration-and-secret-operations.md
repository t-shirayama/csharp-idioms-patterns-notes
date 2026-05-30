# configuration and secret operations

## 概要

設定と secret は、コードよりも環境差分や運用手順に強く影響されます。C# アプリケーションでは `IConfiguration`、Options pattern、環境変数、User Secrets、Key Vault などを組み合わせて扱いますが、どこに保存し、どう注入し、どう検証し、どう漏えいを防ぐかまで考える必要があります。

設定は動作を変える値、secret は漏れると悪用される値です。この 2 つを同じ感覚で扱うと、ログや設定ダンプ、CI 出力から secret が漏れる危険があります。

## 判断基準

- secret はソース管理されるファイルに置かない。
- 設定は型付き Options に束ね、起動時に検証する。
- secret は環境ごとに注入経路を分ける。
- ローテーションが必要な secret は、再起動や再読み込みの方針を決める。
- ログ、例外、health check、diagnostics に secret を出さない。

## 使いどころ

DB connection string、API key、OAuth client secret、storage key、message broker credential、外部サービス endpoint、feature flag、timeout、retry 回数などを扱う場面で使います。特に本番、CI、開発環境で値の入れ方が異なる場合は、Options validation が重要です。

## 良い例

Options を型で表し、起動時に必須値と範囲を検証します。

```csharp
public sealed class PaymentOptions
{
    public required string Endpoint { get; init; }
    public required string ApiKey { get; init; }
    public int TimeoutSeconds { get; init; } = 10;
}

builder.Services
    .AddOptions<PaymentOptions>()
    .Bind(builder.Configuration.GetSection("Payment"))
    .Validate(options => Uri.TryCreate(options.Endpoint, UriKind.Absolute, out _),
        "Payment endpoint must be an absolute URL.")
    .Validate(options => !string.IsNullOrWhiteSpace(options.ApiKey),
        "Payment API key is required.")
    .Validate(options => options.TimeoutSeconds is >= 1 and <= 60,
        "Payment timeout must be between 1 and 60 seconds.")
    .ValidateOnStart();
```

## 避けたい書き方

各所で `IConfiguration` から直接値を読み、secret をログに出すと、設定の契約も漏えい対策も曖昧になります。

```csharp
var apiKey = configuration["Payment:ApiKey"];
logger.LogInformation("Payment API key is {ApiKey}", apiKey);
```

設定が足りない場合も、どの時点で失敗するか分かりにくくなります。

## 改善例

利用側は `IOptions<T>` などを受け取り、secret は値そのものをログに出しません。

```csharp
public sealed class PaymentClient
{
    private readonly PaymentOptions options;
    private readonly ILogger<PaymentClient> logger;

    public PaymentClient(IOptions<PaymentOptions> options, ILogger<PaymentClient> logger)
    {
        this.options = options.Value;
        this.logger = logger;
    }

    public Task SendAsync(CancellationToken cancellationToken)
    {
        logger.LogInformation("Sending payment request to {Endpoint}", options.Endpoint);
        return Task.CompletedTask;
    }
}
```

調査に必要な endpoint や timeout はログに残し、secret は出さないようにします。

## レビュー観点

- secret が repository、設定ファイル、README、テストデータ、CI ログに含まれていないか。
- Options validation で必須値、形式、範囲、相互依存が起動時に検出されるか。
- `IConfiguration` の直接参照が散らばらず、設定の契約が型で表現されているか。
- secret の注入経路が、開発、CI、本番で説明できるか。
- secret のローテーション時に、アプリ再起動や接続再作成の方針があるか。
- ログ、例外、health check、diagnostics に secret や connection string が出ないか。
