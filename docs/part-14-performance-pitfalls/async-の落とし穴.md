# async の落とし穴

## 概要

`async` は I/O 待ちの間にスレッドを占有しないための仕組みです。CPU 処理を自動的に速くするものではありません。sync-over-async、fire-and-forget、キャンセル漏れは、性能問題だけでなく障害時の調査しにくさにもつながります。

```csharp
// 避けたい例: 非同期処理を同期的に待つ。
var response = httpClient.GetAsync(uri).Result;
```

```csharp
var response = await httpClient.GetAsync(uri, cancellationToken);
```

## 判断基準

- I/O-bound な処理は `async` を呼び出しチェーン全体へ伝播させる。
- CPU-bound な処理は `async` では速くならない。必要なら並列化やキューイングとして設計する。
- `CancellationToken` は公開 API から下位の I/O まで渡す。
- fire-and-forget は、例外、キャンセル、終了待ち、リトライ、監視を決められる場合だけ使う。
- 同期完了が多い薄い wrapper では、`async` を付けず `Task` / `ValueTask` を返す方がよい場合がある。

## 使いどころ

- HTTP、DB、ファイル、キューなど待ち時間がある I/O。
- ASP.NET Core や UI アプリで、待機中にスレッドを空けたい処理。
- 呼び出し元からキャンセルやタイムアウトを伝播したい処理。

## 避けたい書き方

- `.Result`、`.Wait()`、`GetAwaiter().GetResult()` で sync-over-async にする。
- `Task.Run` で I/O 待ちを包み、問題を別スレッドへ移すだけにする。
- fire-and-forget で例外、キャンセル、アプリ終了時の扱いを決めない。
- ほぼ同期完了する薄い wrapper に `async` を増やし、意味のない状態機械を作る。
- `CancellationToken` を受け取っているのに、下位 API に渡さない。

```csharp
// 避けたい例: キャンセルが DB 呼び出しまで届かない。
public Task<User?> FindAsync(string email, CancellationToken cancellationToken)
{
    return db.Users.FirstOrDefaultAsync(user => user.Email == email);
}
```

```csharp
public Task<User?> FindAsync(string email, CancellationToken cancellationToken)
{
    return db.Users.FirstOrDefaultAsync(
        user => user.Email == email,
        cancellationToken);
}
```

```csharp
// fire-and-forget が必要なら、背景処理の境界に渡す。
await backgroundQueue.EnqueueAsync(
    new SendReceiptCommand(order.Id),
    cancellationToken);
```

## レビュー観点

- `async` が呼び出しチェーン全体でつながっているか。
- `CancellationToken` が公開 API から下位の I/O まで渡っているか。
- fire-and-forget が必要な場合、監視、ログ、終了待ち、例外処理の置き場があるか。
- CPU-bound な処理と I/O-bound な処理を混同していないか。
- `Task.Run` が ASP.NET Core のリクエスト処理で不要に使われていないか。
- `ConfigureAwait(false)` の要否を、アプリ種別とライブラリ境界で判断しているか。
- `ValueTask` を導入する場合、再 await 不可などの制約を呼び出し側に漏らしていないか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: Task.WhenAll の同時実行数制御

> 対応元: P1 / `Task.WhenAll` の同時実行数制御、timeout、部分失敗、`IAsyncEnumerable` / Channel の backpressure を追加する。

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
