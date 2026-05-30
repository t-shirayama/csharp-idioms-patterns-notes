# allocation を意識する場面

## 概要

allocation はオブジェクトをヒープに確保することです。短命なオブジェクトは GC に回収されますが、高頻度の処理や大量データ処理では、allocation の積み重ねが遅延やメモリ圧迫につながります。

ただし、allocation をすべて避ける必要はありません。通常の業務処理では読みやすさを優先し、リクエストごとに何度も通る処理、ループ内、ログやシリアライズなどの hot path で意識します。

```csharp
// 避けたい例: 大量件数で毎回一時リストを作る。
var activeNames = users
    .Where(user => user.IsActive)
    .Select(user => user.Name)
    .ToList();

foreach (var name in activeNames)
{
    writer.WriteLine(name);
}
```

```csharp
// 必要な場面では、一時コレクションを作らず逐次処理する。
foreach (var user in users)
{
    if (user.IsActive)
    {
        writer.WriteLine(user.Name);
    }
}
```

## 判断基準

- その処理が hot path か、入力件数が増える可能性があるかを先に見る。
- allocation の削減で読みやすさが大きく落ちるなら、測定結果を根拠にする。
- まず不要な `ToList()`、文字列生成、ラムダキャプチャ、一時 DTO を減らす。
- `Span<T>`、`ArrayPool<T>`、object pooling は、所有権と寿命を説明できる場合に使う。
- `struct` 化は allocation を減らす道具だが、コピーコスト、boxing、default 値の扱いも増える。

## 使いどころ

- 大量データを処理するループ、リクエストごとの共通処理、シリアライズ、ログ整形。
- プロファイラやメトリクスで GC 回数、割り当て量、遅延が問題になっている箇所。
- `Span<T>` / `Memory<T>` や pooling を使う前に、まず不要な一時オブジェクトを減らせる箇所。

## 避けたい書き方

- 測定せず、すべてのコードを allocation 回避のために読みにくくする。
- `struct` にすれば速いと決めつけ、大きな値型をコピーし続ける。
- pooling したオブジェクトの返却漏れや、返却後の再利用で状態を壊す。
- キャッシュでメモリを抱え込み、期限切れ、最大件数、無効化の方針を決めない。

```csharp
// 避けたい例: クロージャがループごとに必要になり、意図も追いにくい。
foreach (var customer in customers)
{
    actions.Add(() => SendReminder(customer.Id));
}
```

```csharp
foreach (var customer in customers)
{
    actions.Add(new ReminderCommand(customer.Id));
}
```

## レビュー観点

- 対象コードが本当に hot path か、入力件数が増える可能性があるか。
- ループ内で不要な `ToList()`、文字列生成、ラムダキャプチャが発生していないか。
- `Span<T>`、`ArrayPool<T>`、キャッシュを使う場合、寿命と所有権が明確か。
- 最適化によってテストしにくさや可読性の低下が増えすぎていないか。
- allocation 削減の前後で、BenchmarkDotNet やプロファイラの結果を比較しているか。
- pooling やキャッシュが例外時にも解放、返却、無効化されるか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: ArrayPool<T>

> 対応元: P1 / `ArrayPool<T>`、`Span<T>`、`Memory<T>`、object pooling の所有権、返却、clear、例外時返却の短い例を追加する。

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
