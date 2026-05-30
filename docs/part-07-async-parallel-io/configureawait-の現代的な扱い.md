# ConfigureAwait の現代的な扱い

## 概要

`ConfigureAwait(false)` は、`await` 後に元の同期コンテキストへ戻る必要がないことを示す指定です。古い ASP.NET や UI アプリでは重要な判断点でした。ASP.NET Core では通常、戻るべき同期コンテキストがないため、アプリケーションコードで機械的に付ける必要は薄くなっています。

```csharp
// ライブラリコードでは、呼び出し元のコンテキストに依存しない意図を示すことがある。
var content = await httpClient.GetStringAsync(uri, cancellationToken)
    .ConfigureAwait(false);
```

## 使いどころ

- 汎用ライブラリで、UI スレッドなど呼び出し元のコンテキストに戻る必要がない処理。
- 古い .NET Framework、WPF、WinForms、旧 ASP.NET を含むコードベースの保守。
- チームで方針が決まっており、付与範囲が一貫している場合。

## 判断基準

ASP.NET Core の通常のアプリケーションコードでは、`ConfigureAwait(false)` を付けても可読性以上の効果が小さいことが多いです。プロジェクト方針がないなら、アプリケーション層では機械的に付けず、ライブラリ層や旧 UI / 旧 ASP.NET を含む箇所で必要性を判断します。

WPF や WinForms では、`await` 後に UI コンポーネントへ触るなら元の UI スレッドに戻る必要があります。その直前の `await` に `ConfigureAwait(false)` を付けると、UI 操作で例外や不安定な挙動につながります。

```csharp
// UI 更新が続く処理では、元のコンテキストへ戻る前提を保つ。
var user = await userService.LoadAsync(userId, cancellationToken);
nameTextBox.Text = user.DisplayName;
```

## 避けたい書き方

- ASP.NET Core のアプリケーションコード全体に、理由を確認せず機械的に付ける。
- UI 更新が必要な処理で `ConfigureAwait(false)` を使い、再開後に UI スレッド前提の操作をする。
- 付ける場所と付けない場所が混在し、方針が読み取れない。

```csharp
// 避けたい例: await 後に UI を触るのにコンテキストへ戻らない。
var user = await userService.LoadAsync(userId, cancellationToken)
    .ConfigureAwait(false);

nameTextBox.Text = user.DisplayName;
```

## よくある失敗

- `ConfigureAwait(false)` をデッドロック対策として付けたが、根本原因の `.Result` / `.Wait()` が残っている。
- ライブラリコードの一部だけに付け、await 後の前提がファイルごとに違う。
- `AsyncLocal`、Culture、ログスコープなどの流れまで切れると誤解する。`ConfigureAwait(false)` が抑えるのは主に同期コンテキストへの再開です。
- 性能改善として強調しすぎ、読みやすさやチーム方針を犠牲にする。

## レビュー観点

- 対象がアプリケーションコードか、再利用されるライブラリコードか。
- `await` 後に UI やリクエストコンテキストへ戻る必要があるか。
- プロジェクト内の方針と一貫しているか。
- `ConfigureAwait(false)` の有無を性能改善として過大評価していないか。
- 同期ブロックを残したまま、`ConfigureAwait(false)` だけで問題を隠していないか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: ConfigureAwait(false) と AsyncLocal

> 対応元: P2 / `ConfigureAwait(false)` と `AsyncLocal`、Culture、logging scope が完全に切れるわけではない点を小例で補足する。

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
