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
