# StringBuilder、Span / Memory の入口

## 概要

短い文字列の連結は通常の `+` や文字列補間で十分です。
`StringBuilder`、`Span<T>`、`Memory<T>` は、繰り返し処理や hot path で allocation を減らしたい場面に絞って使います。

最適化系の API は、導入した瞬間にコードの読みやすさを下げることがある。まずは普通に書き、計測やデータ量から必要になった場所にだけ使うのが実務では安定する。

## 使いどころ

- ループで多数の行や断片を組み立てるときは `StringBuilder` を使う。
- 入力文字列の一部を切り出して読むだけなら `ReadOnlySpan<char>` を検討する。
- 非同期境界をまたぐバッファは `Memory<T>`、同期的な一時ビューは `Span<T>` と考える。

```csharp
var builder = new StringBuilder();

foreach (var error in errors)
{
    builder.AppendLine($"- {error.Code}: {error.Message}");
}

return builder.ToString();
```

```csharp
static bool IsHttpUrl(ReadOnlySpan<char> value)
{
    return value.StartsWith("http://", StringComparison.OrdinalIgnoreCase)
        || value.StartsWith("https://", StringComparison.OrdinalIgnoreCase);
}
```

## 判断基準

- 数個の文字列を組み立てるだけなら、文字列補間や `string.Concat` で十分。
- ループで行を積み上げる、ログや CSV を生成するなら `StringBuilder` を検討する。
- substring を大量に作っている hot path では、`ReadOnlySpan<char>` で読むだけにできないか確認する。
- async API や保存が必要なバッファは `Memory<T>`、stack 上の短命な処理は `Span<T>`。
- `Span<T>` を public API に出すと利用者の制約も増えるため、境界では慎重に使う。

`StringBuilder` は初期容量を見積もれるなら指定するとよい。ただし、見積もりが不自然に複雑になるなら、まず読みやすさを優先する。

```csharp
var builder = new StringBuilder(capacity: errors.Count * 32);

foreach (var error in errors)
{
    builder.Append(error.Code)
        .Append(": ")
        .AppendLine(error.Message);
}
```

`Span<T>` は「コピーせずに範囲を見る」ための道具。範囲の寿命が短いことを前提にする。

```csharp
static string GetExtension(ReadOnlySpan<char> fileName)
{
    var index = fileName.LastIndexOf('.');
    return index < 0 ? "" : fileName[(index + 1)..].ToString();
}
```

## 避けたい書き方

- すべての文字列連結を機械的に `StringBuilder` に置き換える。
- 読みやすさを犠牲にして、計測していない箇所に `Span<T>` を広げる。
- `Span<T>` をフィールドに保持しようとする。

```csharp
// 避けたい: ループ内で中間文字列を作り続ける
var text = "";
foreach (var item in items)
{
    text += item.Name + Environment.NewLine;
}
```

```csharp
// 良い例: 行を積み上げる意図が明確
var builder = new StringBuilder();

foreach (var item in items)
{
    builder.AppendLine(item.Name);
}

var text = builder.ToString();
```

`Span<T>` は ref struct なので、通常のフィールド、lambda capture、async メソッドをまたいだ保持などに制約がある。無理に広げると、呼び出し側の設計まで巻き込んでしまう。

```csharp
// 良い例: async 境界をまたぐため Memory を使う
static async Task SaveAsync(ReadOnlyMemory<byte> buffer, Stream stream)
{
    await stream.WriteAsync(buffer);
}
```

`Span<T>` と `Memory<T>` の違いは、寿命と async 境界で判断する。

## テストしやすさ

`StringBuilder` や `Span<T>` を使っても、外から見える仕様は文字列やバイト列としてテストする。最適化のために分岐が増えた場合は、空入力、短い入力、区切り文字なし、境界ちょうどの長さを確認する。

## レビュー観点

- `StringBuilder` を使う理由が、ループや大量データなどの実コストに結びついているか。
- `Span<T>` によって API が読みにくくなっていないか。
- allocation 削減のための変更なら、対象が実際に頻繁に呼ばれる経路か。
- 文字列処理の最適化が、例外処理や入力検証の分かりやすさを壊していないか。
- `Memory<T>` が必要な async 境界に、無理に `Span<T>` を渡そうとしていないか。
- 最適化の前後で culture、文字コード、エラー処理の仕様が変わっていないか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: Span<T> を使う前に Memory<T>

> 対応元: P2 / `Span<T>` を使う前に `Memory<T>`、`ArrayPool<T>`、`string.Create`、通常 API のどれを選ぶかの境界を補う。

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
