# boxing / unboxing

## 概要

boxing は値型を `object` やインターフェイス型として扱うためにヒープ上のオブジェクトへ包む処理です。unboxing はそれを値型へ戻す処理です。単発では小さなコストでも、頻繁に通る処理では hidden allocation になります。

```csharp
// 避けたい例: int が object として boxing される。
var values = new ArrayList();
values.Add(123);

var number = (int)values[0];
```

```csharp
// generic collection なら boxing を避けられ、型安全性も上がる。
var values = new List<int>();
values.Add(123);

var number = values[0];
```

## 判断基準

- 新規コードでは、非 generic collection や `object` ベースの API を避ける。
- 値型を頻繁にログ、メトリクス、フォーマット処理へ渡す箇所は allocation を確認する。
- interface 経由の呼び出し、`Enum`、`params object[]` は boxing の発生源になりやすい。
- boxing 回避のために generic を導入する場合、API の複雑さと利用頻度を比べる。
- 値型を大きくしすぎると、boxing を避けてもコピーコストが増える。

## 使いどころ

- 古い API、非 generic collection、`object` ベースの拡張ポイントを扱うとき。
- ログ、メトリクス、フォーマット処理など、値型を頻繁に渡す箇所。
- 値型がインターフェイス経由で呼ばれる場面の allocation を確認するとき。

## 避けたい書き方

- `ArrayList` や `Hashtable` のような非 generic collection を新規コードで使う。
- `object` 引数の便利メソッドに値型を大量に渡す。
- boxing を避けるためだけに過度に複雑な generic 設計へ寄せる。
- unboxing のキャスト失敗を実行時まで隠す。

```csharp
// 避けたい例: params object[] に値型が入る。
logger.LogInformation("elapsed={Elapsed} count={Count}", elapsed, count);
```

```csharp
// 高頻度な箇所では、型付き API や source-generated logging を検討する。
LogProcessed(logger, elapsed, count);

[LoggerMessage(
    EventId = 10,
    Level = LogLevel.Information,
    Message = "elapsed={Elapsed} count={Count}")]
static partial void LogProcessed(ILogger logger, TimeSpan elapsed, int count);
```

## レビュー観点

- 値型を `object`、`IComparable`、`IFormattable` などへ暗黙変換していないか。
- collection や API が generic で表現できるのに `object` になっていないか。
- boxing の回避が必要なほど高頻度な処理か、測定で確認しているか。
- 型安全性を下げるキャストや unboxing 例外のリスクが残っていないか。
- logger、metrics、serialization の API が hidden allocation を起こしていないか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: enum formatting

> 対応元: P2 / enum formatting、interface constrained generic、`IEquatable<T>`、custom comparer の boxing 回避例を追加する。

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
