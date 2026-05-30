# async / await の基本

## 概要

`async` / `await` は、非同期 API を読みやすい逐次処理の形で書くための仕組みです。`await` した地点で処理はいったん戻り、完了後に続きから再開されます。例外は `Task` に格納され、`await` 時に通常の例外として再送出されます。

```csharp
public async Task<UserProfile> GetProfileAsync(Guid userId, CancellationToken cancellationToken)
{
    var user = await userRepository.FindAsync(userId, cancellationToken);

    if (user is null)
    {
        throw new UserNotFoundException(userId);
    }

    return new UserProfile(user.Id, user.DisplayName);
}
```

## 使いどころ

- DB、HTTP、ファイル、キューなど、完了まで待ち時間が発生する I/O 処理。
- UI や Web サーバーで、待機中にスレッドを占有したくない処理。
- 非同期 API を呼び出す上位 API。途中だけ同期に戻すと詰まりやすい。
- 独立した I/O を並行に待ちたいが、CPU を増やして計算したいわけではない処理。

```csharp
public async Task<UserSummary> GetSummaryAsync(
    Guid userId,
    CancellationToken cancellationToken)
{
    var profileTask = profileClient.GetProfileAsync(userId, cancellationToken);
    var ordersTask = orderRepository.GetRecentOrdersAsync(userId, cancellationToken);

    await Task.WhenAll(profileTask, ordersTask);

    return new UserSummary(await profileTask, await ordersTask);
}
```

## 判断基準

`async` は呼び出し元へ待機を返すための仕組みであり、処理そのものを別スレッドで速くする仕組みではありません。I/O 待ちなら効果がありますが、CPU を使い切る計算では `await` だけでは速くなりません。

非同期メソッドは、原則として上位まで `async` を伝播させます。途中で `.Result` や `.Wait()` に戻すと、例外の包み方、デッドロック、スレッドプール枯渇の原因になります。

## 避けたい書き方

- `Task.Result` や `.Wait()` で非同期処理を同期的に待つ。
- イベントハンドラー以外で `async void` を使う。
- 非同期メソッドなのに `Async` 接尾辞がなく、呼び出し側が待機の必要性を見落とす。
- ループ内で 1 件ずつ `await` し、独立した I/O まで直列化する。

```csharp
// 避けたい例: 例外の扱いが難しく、コンテキスト次第でデッドロックの原因になる。
var profile = profileService.GetProfileAsync(userId, CancellationToken.None).Result;
```

```csharp
// 避けたい例: 各呼び出しが独立していても、常に直列で待ってしまう。
foreach (var id in userIds)
{
    summaries.Add(await userClient.GetSummaryAsync(id, cancellationToken));
}
```

## よくある失敗

- `async` メソッドの中で重い CPU 処理を実行し、リクエスト処理スレッドを長時間使う。
- fire-and-forget で投げた `Task` の例外を誰も観測せず、失敗がログにも出ない。
- `Task.WhenAll` の一部失敗時に、どの入力が失敗したか分からない設計にする。
- キャンセル後も後続処理を続け、呼び出し元が中断したのに外部 I/O が残る。

## レビュー観点

- 呼び出しチェーン全体が `async` でつながっているか。
- `async void` がイベントハンドラーなど妥当な場所に限定されているか。
- 例外を握りつぶさず、呼び出し元が失敗を判断できる形になっているか。
- メソッド名、戻り値、キャンセル引数から非同期処理であることが分かるか。
- 並行実行する場合、同時実行数と失敗時の扱いが決まっているか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: Task.WhenAll の一部失敗時に入力 ID と例外を対応付ける例

> 対応元: P1 / `Task.WhenAll` の一部失敗時に入力 ID と例外を対応付ける例、兄弟タスクのキャンセル方針を追加する。

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
