# 設定と Options pattern

## 概要

設定は `appsettings.json`、環境変数、Secret Manager、Key Vault など複数の供給元から読み込まれます。アプリケーションコードでは `IConfiguration` を直接あちこちで読むより、Options pattern で用途ごとの型に束ねると、設定の意味と検証が明確になります。

`IOptions<T>` は起動時に確定する設定、`IOptionsSnapshot<T>` はリクエスト単位で再評価したい設定、`IOptionsMonitor<T>` は変更通知を受けたい設定で検討します。ただし、動的変更が必要ない設定まで monitor にすると、読み手に余計な期待を与えます。

```csharp
public sealed class MailOptions
{
    public required string SenderAddress { get; init; }
    public required string Endpoint { get; init; }
    public int TimeoutSeconds { get; init; } = 10;
}

builder.Services
    .AddOptions<MailOptions>()
    .Bind(builder.Configuration.GetSection("Mail"))
    .ValidateDataAnnotations()
    .Validate(options => options.TimeoutSeconds > 0, "TimeoutSeconds must be positive")
    .ValidateOnStart();
```

## 使いどころ

- 外部サービスの URL、タイムアウト、機能フラグなど、環境ごとに変わる値を扱う。
- 設定値の名前、型、必須性をコード上で表したい。
- 起動時に設定不備を検出し、実行中の曖昧な失敗を減らしたい。

## 判断基準

設定は「変更される可能性がある値」なら何でも外へ出せばよい、というものではありません。外へ出すほど環境差分や運用手順が増えるため、判断基準を置きます。

- デプロイ環境ごとに変わる値は設定にする。
- 業務ルールとしてコードレビューしたい値は、安易に設定化しない。
- 起動後に変わる必要がない値は `IOptions<T>` を基本にする。
- 変更通知が必要な値だけ `IOptionsMonitor<T>` を使う。
- ユーザー入力や DB 上のマスタ値を Options で代用しない。

```csharp
public sealed class RetryOptions
{
    public int MaxAttempts { get; init; } = 3;
    public TimeSpan Delay { get; init; } = TimeSpan.FromSeconds(1);
}
```

`MaxAttempts` のような値は環境差分として扱いやすい一方、割引率や承認条件のような業務判断は、設定化すると「誰がいつ変更したか」「テストはどこにあるか」が曖昧になりがちです。

## バインド時の注意

Options 型は `required` や nullable reference types だけでは、設定ファイルの欠落を完全には検出できません。`ValidateOnStart()` を使い、アプリケーション起動時に失敗させる方が、実行中に外部 API 呼び出しで失敗するより調査しやすくなります。

```csharp
builder.Services
    .AddOptions<RetryOptions>()
    .BindConfiguration("Retry")
    .Validate(options => options.MaxAttempts is >= 1 and <= 5,
        "MaxAttempts must be between 1 and 5.")
    .Validate(options => options.Delay > TimeSpan.Zero,
        "Delay must be positive.")
    .ValidateOnStart();
```

接続文字列や API キーは Options 型に入れても構いませんが、ログ出力や `ToString()` で漏れないようにします。secret は環境変数、Secret Manager、Key Vault などの供給元で管理し、リポジトリには置きません。

```csharp
public sealed class ExternalApiOptions
{
    public required Uri BaseUri { get; init; }
    public required string ApiKey { get; init; }

    public override string ToString() => $"{nameof(BaseUri)} = {BaseUri}";
}
```

## 避けたい書き方

- サービスや Controller の中で `configuration["Mail:Endpoint"]` のような文字列キーを散らばらせる。
- パスワード、API キー、接続文字列などの secret をリポジトリに置く。
- 設定値を `static` に退避し、テストや環境差し替えを難しくする。
- 「後で変えられるように」と、実際には変更手順もテストもない値を設定化する。

```csharp
// 避けたい例: キー名の変更に弱く、必須設定の検証も遅れる。
var endpoint = configuration["Mail:Endpoint"];
```

```csharp
// 避けたい例: static に逃がすと、テストや並列実行で差し替えにくい。
public static class AppSettings
{
    public static string MailEndpoint { get; set; } = "";
}
```

## テストしやすさ

Options を利用するクラスは、`IOptions<T>` を受け取るより、設定値そのものを受け取る方が単体テストしやすい場合があります。ASP.NET Core の DI 境界では `IOptions<T>` を使い、業務ロジック側には値オブジェクトとして渡す、という分け方も検討できます。

```csharp
public sealed class MailClient
{
    private readonly MailOptions options;

    public MailClient(IOptions<MailOptions> options)
    {
        this.options = options.Value;
    }
}
```

テストでは `Options.Create(new MailOptions { ... })` を使えます。ただし、検証済みであることを前提にするクラスなら、無効な Options を直接渡したテストが本番挙動とずれないか注意します。

## レビュー観点

- 設定が用途ごとの Options 型にまとまっているか。
- 必須値、範囲、形式を起動時に検証しているか。
- secret が設定ファイルに平文で含まれていないか。
- `IOptions` / `IOptionsSnapshot` / `IOptionsMonitor` の選択理由が実際の更新要件と合っているか。
- 設定化した値に、変更手順、検証方法、監査の必要性が考慮されているか。
- Options 型が外部契約として読みやすい名前、単位、既定値を持っているか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: named options

> 対応元: P1 / named options、`IValidateOptions<T>`、`PostConfigure`、reload 時の `IOptionsMonitor` 注意点を追加する。

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
