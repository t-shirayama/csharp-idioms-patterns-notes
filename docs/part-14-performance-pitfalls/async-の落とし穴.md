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
