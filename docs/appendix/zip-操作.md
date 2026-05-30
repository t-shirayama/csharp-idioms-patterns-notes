# ZIP 操作

## 概要

ZIP 操作では `ZipFile` や `ZipArchive` を使って、複数ファイルをまとめて読み書きします。
実務では、一時ディレクトリ、既存ファイルの上書き、巨大ファイル、パス traversal に注意します。

ZIP は単なるファイル集合に見えますが、外部入力として受け取る場合は攻撃面になります。
展開先の正規化、entry 数やサイズの上限、上書き方針、失敗時の片付けを仕様として決めてから実装します。

```csharp
static void ExtractEntrySafely(ZipArchiveEntry entry, string destinationDirectory)
{
    var destinationRoot = Path.GetFullPath(destinationDirectory);
    var destinationPath = Path.GetFullPath(Path.Combine(destinationRoot, entry.FullName));

    if (!destinationPath.StartsWith(destinationRoot + Path.DirectorySeparatorChar, StringComparison.OrdinalIgnoreCase))
    {
        throw new InvalidOperationException($"Invalid ZIP entry path: {entry.FullName}");
    }

    if (string.IsNullOrEmpty(entry.Name))
    {
        Directory.CreateDirectory(destinationPath);
        return;
    }

    Directory.CreateDirectory(Path.GetDirectoryName(destinationPath)!);
    entry.ExtractToFile(destinationPath, overwrite: false);
}
```

## 判断基準

- 信頼できない ZIP は、展開前に entry 数、名前、サイズの上限を確認する。
- 全展開が不要なら、必要な entry だけ stream で読む。
- 上書きが必要な場合は、既存ファイルを直接壊す前に一時ディレクトリへ展開して検証する。
- 巨大ファイルや zip bomb を想定するなら、圧縮後サイズだけでなく展開後サイズも見る。
- パス文字列ではなく `Path.GetFullPath` で正規化してからルート配下か判定する。

## 使いどころ

- 複数ファイルのエクスポート、インポート、バックアップをまとめる。
- ZIP 内の特定 entry だけを読み取り、メモリに載せすぎず処理する。
- 一時ディレクトリを使って、途中失敗時に元データへ影響を出さない。
- ユーザーがアップロードしたテンプレートや成果物を検証してから取り込む。

## よい例

entry の stream を必要な範囲で読み、サイズ上限を越えたら止めます。

```csharp
static async Task<string> ReadSmallTextEntryAsync(ZipArchiveEntry entry, CancellationToken ct)
{
    const long MaxBytes = 64 * 1024;

    if (entry.Length > MaxBytes)
    {
        throw new InvalidOperationException($"ZIP entry is too large: {entry.FullName}");
    }

    await using var stream = entry.Open();
    using var reader = new StreamReader(stream, Encoding.UTF8, detectEncodingFromByteOrderMarks: true);
    return await reader.ReadToEndAsync(ct);
}
```

## 避けたい書き方

- `entry.FullName` をそのままファイルパスとして使う。
- ZIP 全体や entry 全体を安易にメモリへ読み込む。
- 一時ファイルを固定名で作り、並列実行や再実行で衝突させる。
- `overwrite: true` を無条件に使い、既存データを壊す。
- entry 名の文字コード、大小文字、ディレクトリ entry の扱いをテストしない。

```csharp
// 避けたい例: ../ を含む entry で意図しない場所へ書き込める。
foreach (var entry in archive.Entries)
{
    entry.ExtractToFile(Path.Combine(destination, entry.FullName), overwrite: true);
}
```

## テストしやすくする

ZIP のテストは、ファイルシステム境界と検証ロジックを分けると書きやすくなります。
特に `../evil.txt`、空ディレクトリ、同名 entry、サイズ超過、0 byte、UTF-8 以外の名前をテストデータに入れます。

```csharp
static bool IsUnderDirectory(string root, string candidate)
{
    var normalizedRoot = Path.GetFullPath(root);
    var normalizedCandidate = Path.GetFullPath(candidate);

    return normalizedCandidate.StartsWith(
        normalizedRoot + Path.DirectorySeparatorChar,
        StringComparison.OrdinalIgnoreCase);
}
```

## レビュー観点

- zip slip を防ぐため、展開先の正規化とルート配下チェックをしているか。
- サイズ上限、entry 数、上書き可否を仕様として決めているか。
- 失敗時に一時ディレクトリや部分的に作られたファイルが残っても問題ないか。
- 文字コードやパス区切りの差を前提にしたテストがあるか。
- `ZipArchive`、entry stream、出力 stream が確実に dispose されるか。
- ログに entry 名を出す場合、制御文字や長すぎる名前への対策があるか。

---

[Appendix README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: ZIP bomb 対策として entry 数

> 対応元: P0 / ZIP bomb 対策として entry 数、総展開サイズ、読み取り時の byte count、同名・大小文字衝突、Windows 予約名、絶対パス、symlink 相当、temp 展開後の atomic move を追加する。

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
