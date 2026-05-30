# metrics / tracing / OpenTelemetry

## 概要

観測性では、logs、metrics、traces を役割で分けます。ログは個別イベントの説明、metrics は時間軸での傾向、traces は一つの処理がどの依存先や内部処理を通ったかを見るためのものです。

OpenTelemetry は、この 3 つを共通の考え方で収集するための基盤です。ただし、導入しただけでは十分ではありません。span 名、tag、metric label、sampling、例外の記録方法が悪いと、データ量だけが増えて調査に使えない状態になります。

## 判断基準

- 障害調査で「何が起きたか」を見るなら log。
- 「どれくらい起きているか」「いつから悪化したか」を見るなら metric。
- 「このリクエストはどこで遅くなったか」を見るなら trace。
- metric label には低 cardinality な値だけを入れる。
- trace tag には調査に必要な識別子を入れるが、機密情報や巨大な値は入れない。

## 使いどころ

Web API、バッチ、メッセージ処理、外部 HTTP 呼び出し、DB アクセスなど、失敗や遅延が利用者影響に直結する場所で使います。特に timeout、retry、cancellation、依存先エラーは、ログだけでなく metric と trace にも残すと切り分けやすくなります。

## 良い例

処理時間を metric として記録し、trace には調査に必要な低リスクの属性だけを付けます。

```csharp
using System.Diagnostics;
using System.Diagnostics.Metrics;

static readonly ActivitySource ActivitySource = new("OrderService");
static readonly Meter Meter = new("OrderService");
static readonly Histogram<double> DurationMs =
    Meter.CreateHistogram<double>("orders.create.duration_ms");

public async Task CreateOrderAsync(CreateOrder command, CancellationToken cancellationToken)
{
    using var activity = ActivitySource.StartActivity("orders.create");
    activity?.SetTag("order.type", command.Type);

    var started = Stopwatch.GetTimestamp();

    try
    {
        await repository.SaveAsync(command, cancellationToken);
        activity?.SetStatus(ActivityStatusCode.Ok);
    }
    catch (Exception ex)
    {
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        throw;
    }
    finally
    {
        var elapsed = Stopwatch.GetElapsedTime(started).TotalMilliseconds;
        DurationMs.Record(elapsed, new KeyValuePair<string, object?>("order.type", command.Type));
    }
}
```

## 避けたい書き方

高 cardinality な値を metric label に入れると、時系列データが爆発します。

```csharp
DurationMs.Record(
    elapsed,
    new KeyValuePair<string, object?>("user.id", command.UserId),
    new KeyValuePair<string, object?>("order.id", command.OrderId));
```

`user.id` や `order.id` は値の種類が多すぎるため、metric label には向きません。必要なら trace や構造化ログで扱います。

## 改善例

metric は集計軸を少数に絞り、個別調査用の ID は trace に寄せます。

```csharp
activity?.SetTag("order.id", command.OrderId);

DurationMs.Record(
    elapsed,
    new KeyValuePair<string, object?>("order.type", command.Type),
    new KeyValuePair<string, object?>("result", "success"));
```

metric で「どの種類の注文が遅いか」を見て、個別の遅いリクエストは trace で追う、という分担にします。

## レビュー観点

- metric 名、span 名、tag 名が、実装者だけでなく運用者にも分かる名前か。
- 高 cardinality な値や機密情報を metric label に入れていないか。
- timeout、retry、cancellation、validation error、dependency error を区別できるか。
- 例外が trace に error として記録され、ログにも調査に必要な文脈があるか。
- sampling しても重要な失敗が見えなくならないか。
- dashboard や alert で使う metric が、利用者影響を表しているか。

<!-- TODO対応追記 -->

## TODO対応: ASP.NET Core での OpenTelemetry 最小構成

> 対応元: P0 / ASP.NET Core での OpenTelemetry 最小構成、`ResourceBuilder`、exporter、collector、trace-log correlation、semantic conventions の例を追加する。

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


## TODO対応: SLI/SLO

> 対応元: P0 / SLI/SLO、alert、dashboard、runbook までの設計観点を追加する。

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
