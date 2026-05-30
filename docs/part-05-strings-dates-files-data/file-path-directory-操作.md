# file / path / directory 操作

## 概要

ファイル操作では、パス結合、文字コード、例外、リソース解放を明示します。
OS 差や権限、競合、巨大ファイルを想定しないコードは、開発環境では動いても運用で壊れやすくなります。

ファイルシステムは常に外部依存です。存在確認の直後に消える、権限が変わる、別プロセスが掴む、パス区切りが OS で違う、といった前提で書く。正常系だけ短いコードほど、運用時の失敗が見えにくい。

## 使いどころ

- パス結合には文字列連結ではなく `Path.Combine` を使う。
- 小さなファイルの読み書きは `File.ReadAllTextAsync` などで簡潔に書く。
- 大きなファイルや逐次処理では `Stream` を使う。
- 一時ファイルは `Path.GetTempPath` や `Path.GetRandomFileName` を組み合わせる。

```csharp
var filePath = Path.Combine(baseDirectory, "exports", $"{date:yyyyMMdd}.csv");

await File.WriteAllTextAsync(
    filePath,
    content,
    Encoding.UTF8,
    cancellationToken);
```

```csharp
await using var stream = File.OpenRead(filePath);
using var reader = new StreamReader(stream, Encoding.UTF8);

while (await reader.ReadLineAsync(cancellationToken) is { } line)
{
    ProcessLine(line);
}
```

## 判断基準

- 小さく、全体を一度に扱ってよいファイルなら `File.ReadAllTextAsync` / `WriteAllTextAsync`。
- 大きい、逐次処理したい、途中でキャンセルしたいなら `Stream` と reader / writer。
- パスは `Path.Combine`、`Path.Join`、`Path.GetFullPath` を使い、区切り文字を直接書かない。
- ユーザー入力を含むパスは、許可した base directory の内側に収まるか検証する。
- 文字コードは `Encoding.UTF8` など明示する。BOM の有無が仕様に影響する場合は特に確認する。

ユーザー指定のファイル名を使う場合は、正規化したフルパスが許可ディレクトリ配下かを確認する。

```csharp
var root = Path.GetFullPath(baseDirectory);
var rootWithSeparator = Path.EndsInDirectorySeparator(root)
    ? root
    : root + Path.DirectorySeparatorChar;
var fullPath = Path.GetFullPath(Path.Combine(root, fileName));

if (!fullPath.StartsWith(rootWithSeparator, StringComparison.OrdinalIgnoreCase))
{
    throw new InvalidOperationException("Invalid path.");
}
```

存在確認してから開くコードは、競合に弱い。最終的には open / read / write の例外を扱う。

```csharp
try
{
    await using var stream = File.OpenRead(filePath);
    await ImportAsync(stream, cancellationToken);
}
catch (FileNotFoundException)
{
    return ImportResult.NotFound(filePath);
}
catch (UnauthorizedAccessException)
{
    return ImportResult.AccessDenied(filePath);
}
```

## 避けたい書き方

- `basePath + "\\" + fileName` のように区切り文字を直接書く。
- ユーザー入力をそのままファイルパスに使う。
- `File.ReadAllText` で巨大ファイルを無条件に読み込む。
- `Stream` を `using` せず、例外時に閉じ忘れる。

```csharp
// 避けたい例: パストラバーサルと OS 差に弱い
var path = uploadDirectory + "\\" + request.FileName;
await File.WriteAllBytesAsync(path, request.Content);
```

```csharp
// 良い例: ファイル名だけを取り出し、保存先を制御する
var safeName = Path.GetFileName(request.FileName);
var path = Path.Combine(uploadDirectory, safeName);

await File.WriteAllBytesAsync(path, request.Content, cancellationToken);
```

`Path.GetFileName` だけで全ての安全性が保証されるわけではない。拡張子、サイズ、上書き可否、保存先の権限も別途確認する。

## テストしやすさ

ファイル操作は、ロジックと I/O を分けるとテストしやすい。パスを組み立てる処理、行を解釈する処理、実際にファイルを読む処理を分ける。統合テストでは temporary directory を使い、存在しないファイル、空ファイル、巨大に近い入力、権限エラー相当の例外を確認する。

## レビュー観点

- パス traversal を防いでいるか。
- ファイルが存在しない、権限がない、別プロセスが使用中といった失敗を想定しているか。
- 同期 I/O が ASP.NET Core などのリクエスト処理を塞いでいないか。
- 文字コードと改行コードの前提が明示されているか。
- `File.Exists` の結果を過信せず、実際の I/O 例外を扱っているか。
- ユーザー入力から作るファイル名に、許可拡張子、サイズ、上書きルールがあるか。
- `CancellationToken` を非同期 I/O や長い処理に渡しているか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: パストラバーサル対策を

> 対応元: P0 / パストラバーサル対策を、OS ごとの大文字小文字、区切り文字、シンボリックリンク・junction・hard link まで含めて拡充する。

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


## TODO対応: 一時ファイル

> 対応元: P0 / 一時ファイル、原子的な置き換え、`FileMode.CreateNew`、部分書き込み、上書き競合、後始末の例を追加する。

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
