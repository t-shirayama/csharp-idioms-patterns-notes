# Task、ValueTask、CancellationToken

## 概要

`Task` は非同期処理の完了、失敗、キャンセルを表す基本型です。戻り値がある場合は `Task<T>` を使います。`ValueTask` は割り当て削減のための型ですが、扱いに制約があるため、通常のアプリケーションコードでは `Task` を第一候補にします。

`CancellationToken` は「呼び出し元が処理をやめたい」という意思を伝えるためのものです。タイムアウトとは似ていますが、責務が異なります。キャンセルは呼び出し元の都合、タイムアウトは処理や外部依存の許容時間です。

```csharp
public async Task<IReadOnlyList<Order>> SearchAsync(
    OrderSearchCondition condition,
    CancellationToken cancellationToken)
{
    cancellationToken.ThrowIfCancellationRequested();

    return await orderRepository.SearchAsync(condition, cancellationToken);
}
```

## 使いどころ

- 公開する非同期 API は、原則として `Task` または `Task<T>` を返す。
- UI 操作の中断、HTTP リクエストの中断、バッチ停止などを下位処理へ伝播する。
- 高頻度で同期完了する低レイヤー API では、測定したうえで `ValueTask` を検討する。
- キャンセル可能な API は、待機、リトライ、DB、HTTP、ファイル I/O のすべてに token を渡す。

```csharp
public async Task ImportAsync(
    IAsyncEnumerable<OrderRow> rows,
    CancellationToken cancellationToken)
{
    await foreach (var row in rows.WithCancellation(cancellationToken))
    {
        await orderRepository.SaveAsync(row.ToOrder(), cancellationToken);
    }
}
```

## 判断基準

アプリケーションコードの戻り値は、まず `Task` / `Task<T>` を選びます。`ValueTask` は「同期完了が非常に多い」「割り当てがボトルネック」「呼び出し規約を守れる」ことが分かってから検討します。`ValueTask` は複数回 await できないケースがあり、保存して後で待つような使い方にも向きません。

`CancellationToken` は省略可能に見えても、公開 API では受け取れる形にしておくと後から効きます。ただし、更新処理の途中でキャンセルされたときに整合性が壊れる場合は、キャンセルを受ける境界を慎重に置きます。

## 避けたい書き方

- `CancellationToken.None` を下位処理へ固定で渡し、呼び出し元の中断要求を無視する。
- `ValueTask` を「新しいから」という理由だけで使う。
- キャンセルを例外と同じ障害としてログに大量出力する。
- `OperationCanceledException` を広い `catch (Exception)` で捕まえて通常の失敗として扱う。

```csharp
// 避けたい例: 上位のキャンセル要求が DB 呼び出しまで届かない。
return await dbContext.Orders.ToListAsync(CancellationToken.None);
```

```csharp
// 避けたい例: キャンセルを握りつぶして成功扱いに見せてしまう。
try
{
    await sender.SendAsync(message, cancellationToken);
}
catch (Exception)
{
    return SendResult.Failed;
}
```

## よくある失敗

- token を受け取っているが、`Task.Delay`、`ReadFromJsonAsync`、`ToListAsync` などへ渡していない。
- `CancellationTokenSource` を作ったあと dispose せず、タイマーや登録が長く残る。
- キャンセルをログの Error として扱い、運用時の障害ノイズを増やす。
- `ValueTask` を共通 interface に出し、呼び出し側の扱いを難しくする。

## レビュー観点

- 非同期メソッドの引数に必要な `CancellationToken` があり、下位 API に渡しているか。
- キャンセルとタイムアウトと業務エラーを区別して扱っているか。
- `ValueTask` の使用に、実測または明確な低レイヤー上の理由があるか。
- キャンセル時に中途半端な状態が残らないよう、更新処理の境界が整理されているか。
- `catch` の条件でキャンセルまでリトライやエラー変換の対象にしていないか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: ValueTask の複数 await 禁止

> 対応元: P1 / `ValueTask` の複数 await 禁止、保存して後で await しない、`AsTask()` の判断例を追加する。

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
