# exception、using、IDisposable、リソース管理

## 概要

例外は想定外の失敗を伝える仕組みであり、ファイル、ストリーム、DB 接続などのリソースは確実に解放する必要があります。`using` statement、using declaration、`IDisposable`、`IAsyncDisposable` を正しく使うと、正常系と異常系の両方でリークを防げます。

- 例外は握りつぶさず、必要な文脈を足して上位に渡す
- `using` は `Dispose` を確実に呼ぶために使う
- 非同期に解放が必要な型では `await using` を使う
- リソース所有者が誰かを明確にする

## 判断基準

例外にするか戻り値で表すかは、「呼び出し側が通常の分岐として扱うべきか」で決めます。検索結果なし、入力 validation 失敗、在庫不足のように業務フローで起きることは、戻り値、Result 型、validation error として表す方が扱いやすい場合があります。ファイル破損、DB 接続失敗、外部 API の予期しない応答のように処理を続けられない失敗は例外が自然です。

リソース管理では「作った側が破棄する」を基本にします。メソッド引数で受け取った `Stream` や `HttpClient` を内部で破棄すると、呼び出し元の再利用前提を壊すことがあります。

`IDisposable` と `IAsyncDisposable` の違いは、解放処理が非同期 API を必要とするかです。DB 接続、非同期ストリーム、バッファ flush などで非同期解放が提供されているなら `await using` を検討します。同期 `using` で済ませられる型にまで無理に `IAsyncDisposable` を広げる必要はありません。

using declaration (`using var`) はスコープの終わりで破棄されます。短く書けますが、メソッド全体の末尾まで生きると困るリソースでは、ブロック付きの `using` で寿命を狭める方が意図が伝わります。

## 使いどころ

ファイルやストリームを扱う処理では、短いスコープで `using` を使います。呼び出し元から渡された stream をメソッド内で勝手に dispose するかどうかは、API の契約として明確にします。

```csharp
public static async Task<string> ReadAllTextAsync(
    string path,
    CancellationToken cancellationToken)
{
    await using var stream = File.OpenRead(path);
    using var reader = new StreamReader(stream);

    return await reader.ReadToEndAsync(cancellationToken);
}
```

業務上の失敗とプログラム上の異常は分けます。入力不正など予測できる失敗は validation error として返す設計も検討し、DB 接続失敗やファイル破損のような予期しない失敗は例外として扱います。

```csharp
await using var transaction = await db.Database.BeginTransactionAsync(cancellationToken);

await SaveAsync(order, cancellationToken);
await transaction.CommitAsync(cancellationToken);
```

## 良い例

例外を捕捉するなら、回復する、文脈を足して投げ直す、境界でログに残す、のいずれかの目的を明確にします。`throw;` はスタックトレースを保ちますが、`throw ex;` は原因追跡を難しくします。

```csharp
try
{
    await repository.SaveAsync(order, cancellationToken);
}
catch (DbUpdateException ex)
{
    throw new OrderSaveException(order.Id, "注文の保存に失敗しました。", ex);
}
```

`IDisposable` を実装する型では、所有しているリソースだけを破棄します。DI コンテナから渡された依存や共有 `HttpClient` のように、自分が生成していないものは勝手に dispose しない設計にします。

```csharp
public sealed class ReportWriter : IDisposable
{
    private readonly StreamWriter _writer;

    public ReportWriter(string path)
    {
        _writer = File.CreateText(path);
    }

    public void WriteLine(string line) => _writer.WriteLine(line);

    public void Dispose() => _writer.Dispose();
}
```

## 避けたい書き方

例外を空の `catch` で握りつぶすと、障害調査が困難になります。ログを出す場合も、同じ例外を何層でもログに出すとノイズになるため、境界で一度記録する方が扱いやすいことが多いです。

```csharp
// 避けたい例: 失敗した事実が呼び出し元にも運用にも伝わらない
try
{
    SaveReport(report);
}
catch
{
}
```

リソース解放を `finally` で手書きするより、まず `using` で表せないかを考えます。制御フローが増えるほど、例外時の抜け漏れが起きやすくなります。

```csharp
// 避けたい例: 呼び出し元が所有する stream を勝手に閉じている
public static void WriteCsv(Stream output, IEnumerable<Row> rows)
{
    using var writer = new StreamWriter(output);
    WriteRows(writer, rows);
}
```

## 改善例

業務的に想定できる失敗は、呼び出し側が分岐しやすい形に寄せます。例外で制御フローを書かないことで、テストケースも「成功」「失敗理由」の単位で置きやすくなります。

```csharp
public sealed record ReserveResult(bool Succeeded, string? Reason);

public ReserveResult Reserve(Inventory inventory, int quantity)
{
    if (inventory.Available < quantity)
    {
        return new ReserveResult(false, "在庫が不足しています。");
    }

    inventory.Reserve(quantity);
    return new ReserveResult(true, null);
}
```

引数で受け取ったリソースを閉じない契約にするなら、`leaveOpen: true` などを明示します。API コメントやメソッド名だけに頼らず、コード上でも所有権が分かる形にします。

```csharp
public static void WriteCsv(Stream output, IEnumerable<Row> rows)
{
    using var writer = new StreamWriter(output, leaveOpen: true);
    WriteRows(writer, rows);
}
```

## レビュー観点

- `IDisposable` / `IAsyncDisposable` な型を適切なスコープで解放しているか
- 例外を握りつぶしていないか
- 例外メッセージに調査に必要な文脈があるか。ただし secret や個人情報を含めていないか
- 呼び出し元から受け取ったリソースの所有権を勝手に奪っていないか
- 予測できる業務エラーと予期しない例外を混ぜていないか
- catch した例外に対して、回復、変換、ログ、再スローの目的が説明できるか
- 非同期リソースに `using` ではなく `await using` が必要ではないか
- `using var` の寿命が長すぎず、必要な範囲でリソースを閉じているか

## テスト観点

- 正常系だけでなく、途中で例外が起きた場合にも所有リソースが dispose されるか
- 呼び出し元から渡された `Stream` などが、メソッド終了後も契約どおり利用可能か
- 業務エラーが例外ではなく Result / validation error として呼び出し側で分岐できるか
- 例外変換時に inner exception と調査に必要な文脈が残るか

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: cancellation と例外を混ぜない判断

> 対応元: P1 / cancellation と例外を混ぜない判断、exception filter、`OperationCanceledException`、async dispose 失敗時の扱いを追加する。

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
