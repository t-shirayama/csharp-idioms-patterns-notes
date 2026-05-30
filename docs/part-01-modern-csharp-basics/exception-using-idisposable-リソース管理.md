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

## レビュー観点

- `IDisposable` / `IAsyncDisposable` な型を適切なスコープで解放しているか
- 例外を握りつぶしていないか
- 例外メッセージに調査に必要な文脈があるか。ただし secret や個人情報を含めていないか
- 呼び出し元から受け取ったリソースの所有権を勝手に奪っていないか
- 予測できる業務エラーと予期しない例外を混ぜていないか
- catch した例外に対して、回復、変換、ログ、再スローの目的が説明できるか
- 非同期リソースに `using` ではなく `await using` が必要ではないか

---

[Part README に戻る](README.md)


