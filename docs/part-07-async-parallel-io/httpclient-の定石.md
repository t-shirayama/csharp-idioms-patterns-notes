# HttpClient の定石

## 概要

`HttpClient` は外部 I/O の代表例です。実務では、再利用、タイムアウト、キャンセル、リトライ、JSON 変換、エラーレスポンスの扱いをまとめて設計します。特に `HttpClient` をリクエストごとに `new` する書き方は避け、ASP.NET Core では `IHttpClientFactory` を使うのが基本です。

```csharp
public sealed class PaymentClient(HttpClient httpClient)
{
    public async Task<PaymentResult> GetResultAsync(
        string paymentId,
        CancellationToken cancellationToken)
    {
        using var response = await httpClient.GetAsync(
            $"/payments/{paymentId}",
            cancellationToken);

        if (response.StatusCode == HttpStatusCode.NotFound)
        {
            return PaymentResult.NotFound(paymentId);
        }

        response.EnsureSuccessStatusCode();

        var result = await response.Content.ReadFromJsonAsync<PaymentResult>(
            cancellationToken: cancellationToken);

        return result ?? throw new InvalidOperationException("Empty payment response.");
    }
}
```

## 使いどころ

- 外部 API 呼び出しを専用クライアントクラスに閉じ込める。
- `IHttpClientFactory` で接続管理、ベース URL、既定ヘッダー、ポリシーを構成する。
- エラーレスポンスを業務上の結果として扱うか、例外として扱うかを境界で決める。
- 認証ヘッダー、相関 ID、タイムアウト、JSON オプションを呼び出し側へ散らばらせたくない。

```csharp
builder.Services.AddHttpClient<PaymentClient>(client =>
{
    client.BaseAddress = new Uri("https://payments.example.com");
    client.Timeout = TimeSpan.FromSeconds(10);
});
```

## 判断基準

外部 API は、呼び出し元から見ると「業務上の結果を返す依存」です。HTTP ステータス、ヘッダー、JSON の細部は専用クライアントに閉じ込め、アプリケーション層には `PaymentResult.NotFound` や `PaymentResult.Accepted` のような意味のある結果を返します。

`HttpClient.Timeout` は 1 回の HTTP 呼び出しの待機上限で、呼び出し元の `CancellationToken` はユーザー操作やリクエスト中断の意思です。リトライを組み合わせる場合は、1 回ごとの timeout と全体 timeout を分けて考えます。

## 避けたい書き方

- 呼び出しごとに `new HttpClient()` する。
- すべての非 2xx を `EnsureSuccessStatusCode()` だけで処理し、404 や 409 など業務判断に必要な応答を失う。
- `Timeout`、リトライ、`CancellationToken` の責務が重なり、どこで中断されたか分からない。
- `BaseAddress` や相対 URL の組み立てを呼び出し側に散らばらせる。

```csharp
// 避けたい例: 接続管理や設定の一貫性を壊しやすい。
using var client = new HttpClient();
var json = await client.GetStringAsync(url);
```

```csharp
// 避けたい例: ステータスコードの意味が呼び出し側へ漏れている。
var response = await httpClient.GetAsync(url, cancellationToken);
if (response.StatusCode == HttpStatusCode.NotFound)
{
    return null;
}
```

## よくある失敗

- `HttpResponseMessage` や `Stream` を dispose せず、接続を長く占有する。
- エラーレスポンス本文を読まずに捨て、障害調査に必要な情報がログに残らない。
- JSON のフィールド追加、null、enum の未知値など、外部 API の互換性を想定していない。
- 429 や 503 をリトライするが、`Retry-After` やバックオフを見ていない。

## レビュー観点

- `HttpClient` の生成とライフタイムが管理されているか。
- HTTP ステータスごとの扱いが、業務要件と一致しているか。
- JSON の null、形式不正、互換性のない変更を考慮しているか。
- キャンセルとタイムアウトが呼び出し元から追跡できるか。
- 外部 API の仕様変更を受け止める DTO とドメインモデルの境界があるか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: 429/503

> 対応元: P0 / 429/503、`Retry-After`、指数バックオフ、jitter、冪等性キーを含む retry policy の例を追加する。

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


## TODO対応: 大きなレスポンス向けに HttpCompletionOption.ResponseH…

> 対応元: P1 / 大きなレスポンス向けに `HttpCompletionOption.ResponseHeadersRead`、stream dispose、全量読み込み回避を追加する。

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
