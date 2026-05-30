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
