# 補足トピックのレビュー checklist

プロセス起動、環境変数、バージョン情報、ZIP、reflection、source generator、unsafe / native interop は、コード量のわりに障害時の影響範囲が広くなりやすい領域です。
この checklist は「動くか」だけでなく、入力の境界、失敗時の観測性、テストしやすさ、将来の保守範囲をレビューで確認するためのものです。

## 使い方

- 変更差分の中で、外部プロセス、環境、ファイル、生成コード、実行時型情報、native 境界に触れている箇所を先に拾う。
- 各項目は「説明できるか」「テストできるか」「失敗時に分かるか」の順で確認する。
- すべてを満たすことより、暗黙の前提が残っていないかを見る。前提が妥当なら、コメント、テスト名、運用手順のどこかに残す。

## 全体で見ること

- 外部入力、ファイルパス、環境変数、プロセス引数を信頼しすぎていないか。
- 例外、戻り値、ログ、diagnostic、終了コードのどれで失敗に気づけるかが明確か。
- OS、CPU アーキテクチャ、カルチャ、文字コード、改行コード、大文字小文字の差分を想定しているか。
- 単体テストでは境界を差し替え、結合テストでは実環境に近い失敗を確認できるか。
- secret、個人情報、内部パスがログ、例外、snapshot、生成ファイルに出ていないか。
- 「便利だから使う」ではなく、その仕組みが必要な理由と削除できる条件を説明できるか。

## プロセス起動

### レビューで見ること

- `ProcessStartInfo.ArgumentList` を使い、引数を手で連結していないか。
- shell が必要な場面だけ `UseShellExecute = true` にしているか。
- 実行ファイルのパスを、PATH 依存にするのか、明示パスにするのか決めているか。
- `WorkingDirectory`、環境変数、文字コード、標準入力の有無を呼び出し元の暗黙状態に依存させていないか。
- 標準出力と標準エラーを両方読み、バッファ詰まりによる deadlock を避けているか。
- タイムアウト、キャンセル、終了コード、stderr の扱いが呼び出し側に伝わる形になっているか。
- キャンセル時に子プロセスを残さないか。必要なら process tree ごと終了する方針があるか。
- ログに出すコマンドラインから secret や token を伏せているか。

### 見落としやすい観点

- Windows と Linux で引用符、拡張子、実行権限、改行、標準出力の encoding が違う。
- `Process.Start(fileName, arguments)` は一見短いが、引数の quoting をレビューしにくい。
- `WaitForExit` だけで stdout / stderr を後から読むと、出力量によって止まることがある。
- 終了コード 0 でも stderr に警告が出るツールがある。警告を失敗扱いにするかは要件で決める。
- テストでは本物の `git` や `dotnet` を呼ぶ前に、薄い wrapper を差し替えられるかを見る。

避けたい例:

```csharp
var process = Process.Start("git", $"commit -m \"{message}\"");
```

確認したい例:

```csharp
var startInfo = new ProcessStartInfo("git")
{
    RedirectStandardOutput = true,
    RedirectStandardError = true,
    UseShellExecute = false
};
startInfo.ArgumentList.Add("commit");
startInfo.ArgumentList.Add("-m");
startInfo.ArgumentList.Add(message);
```

## 環境変数

### レビューで見ること

- 読み取り箇所が散らばらず、設定オブジェクトや境界クラスに集約されているか。
- 未設定、空文字、空白、不正値を区別して扱っているか。
- `int.Parse` や `bool.Parse` で起動時に落とすのか、既定値に戻すのかが要件として決まっているか。
- `TimeSpan`、数値、列挙値、URL、パスの parse で culture や単位を明示しているか。
- process / user / machine のスコープ差を意識しているか。
- Linux では名前の大文字小文字が区別される前提で、環境変数名の揺れを増やしていないか。
- secret をログ、例外メッセージ、テスト snapshot、diagnostic dump に出していないか。
- テストで環境変数を書き換える場合、並列実行や後始末に配慮しているか。

### 見落としやすい観点

- `Environment.GetEnvironmentVariable` は呼ぶたびに現在値を見る。起動時固定にしたい設定は読み取り時点を決める。
- 空文字は「明示的に無効化」として使われることがある。未設定と同じにしてよいか確認する。
- CI、コンテナ、Windows サービス、IIS、systemd では設定の注入経路が違う。
- 本番だけ machine scope に残った古い値を読む事故がある。優先順位を明文化する。
- テストで global な環境を触るなら、fixture で復元し、並列化を止める必要がある。

避けたい例:

```csharp
var timeout = int.Parse(Environment.GetEnvironmentVariable("APP_TIMEOUT_SECONDS")!);
```

確認したい例:

```csharp
var raw = Environment.GetEnvironmentVariable("APP_TIMEOUT_SECONDS");
var timeout = int.TryParse(raw, NumberStyles.None, CultureInfo.InvariantCulture, out var seconds)
    ? TimeSpan.FromSeconds(seconds)
    : TimeSpan.FromSeconds(30);
```

## バージョン情報

### レビューで見ること

- 表示用バージョン、互換性判定、ファイルバージョン、protocol version を混同していないか。
- prerelease、build metadata、0 埋め、桁数差を文字列比較していないか。
- assembly metadata、NuGet package version、外部 API version の責務が分かれているか。
- ログや問い合わせ対応で必要なバージョン情報を起動時に確認できるか。
- バージョン差による分岐を、古い互換コードとして残しすぎていないか。

### 見落としやすい観点

- `System.Version` は SemVer の prerelease や build metadata を表現しない。
- `AssemblyInformationalVersion` は表示や診断向きで、数値比較向きとは限らない。
- plugin や外部サービスと合わせる version は、アプリ本体の version と別に管理した方が読みやすい。

避けたい例:

```csharp
if (currentVersion >= "2.10.0")
{
    UseNewFormat();
}
```

確認したい例:

```csharp
if (Version.TryParse(currentVersion, out var version)
    && version >= new Version(2, 10, 0))
{
    UseNewFormat();
}
```

## ZIP

### レビューで見ること

- 展開先パスが zip slip を起こさないよう、正規化後にルート配下か確認しているか。
- 絶対パス、`..`、区切り文字の混在、末尾スペース、Windows の予約名を拒否または扱いを決めているか。
- 同名ファイル、大小文字だけ違う名前、ディレクトリエントリ、空ファイルを扱えるか。
- 上書きを許すか、既存ファイルがある場合に失敗させるかが明確か。
- 圧縮率が極端な入力、大きすぎる entry、大量 entry でメモリとディスクを使いすぎないか。
- 文字コード、タイムスタンプ、パーミッション、実行ビットの扱いを要件に合わせているか。
- 展開途中で失敗した場合、部分的なファイルを削除するか、隔離して再試行可能にするか決めているか。
- ZIP 内のファイル種別を信用せず、必要なら拡張子、MIME、magic number を確認しているか。

### 見落としやすい観点

- `ExtractToDirectory` は短いが、上書き、サイズ制限、entry ごとの検査を差し込みにくい。
- case-insensitive なファイルシステムでは、`a.txt` と `A.txt` が衝突する。
- `entry.FullName` がディレクトリを表すかどうかは、末尾の `/` だけに頼ると壊れやすい。
- 先に一時ディレクトリへ展開して検査し、最後に移動すると、途中失敗の影響範囲を狭めやすい。
- アップロード ZIP は正常系サイズだけでなく、最大件数、最大展開後サイズ、最大パス長をテストする。

避けたい例:

```csharp
ZipFile.ExtractToDirectory(zipPath, destinationPath);
```

確認したい例:

```csharp
var root = Path.GetFullPath(destinationPath);

foreach (var entry in archive.Entries)
{
    var target = Path.GetFullPath(Path.Combine(root, entry.FullName));
    var relative = Path.GetRelativePath(root, target);

    if (relative == ".."
        || relative.StartsWith($"..{Path.DirectorySeparatorChar}", StringComparison.Ordinal)
        || Path.IsPathRooted(relative))
    {
        throw new InvalidOperationException("ZIP entry is outside the destination.");
    }
}
```

## reflection

### レビューで見ること

- compile-time に解決できるものまで reflection にしていないか。
- `BindingFlags`、null 戻り値、overload、generic、indexer、non-public member の扱いが明示されているか。
- 型名や member 名を外部入力から受ける場合、許可リストや対象 assembly の境界があるか。
- trimming、AOT、難読化、名前変更で壊れる前提を持っているか。
- 実行時例外を握りつぶさず、対象型、member 名、binding 条件を診断できるか。
- hot path で使う場合、キャッシュや delegate 化の必要性を測定して判断しているか。
- private member へのアクセスを、テスト都合や一時回避として常態化させていないか。

### 見落としやすい観点

- `GetProperty(name)` は overload や indexer を意識しないと、意図と違う member を拾うことがある。
- `Type.GetType(string)` は assembly-qualified name でないと見つからない場合がある。
- nullable annotation は reflection だけでは読み取りづらい。契約として必要なら別の表現を検討する。
- trimming 対象では、reflection で参照される member が削られないよう属性や設計で守る必要がある。
- reflection の失敗を「設定ミス」として扱うのか「プログラムの不具合」として扱うのかで、例外型やログの粒度が変わる。

避けたい例:

```csharp
var value = obj.GetType().GetProperty(name)!.GetValue(obj);
```

確認したい例:

```csharp
var property = obj.GetType().GetProperty(
    name,
    BindingFlags.Instance | BindingFlags.Public);

if (property is null)
{
    throw new InvalidOperationException($"Property was not found: {name}");
}
```

## source generator

### レビューで見ること

- generator がなくても通常のビルドエラーとして原因を追えるか。
- 生成対象の入力が syntax、semantic model、AdditionalFiles、AnalyzerConfigOptions のどれに依存するか整理されているか。
- incremental generator の cache が効く粒度で実装されているか。
- `CancellationToken` を無視して重い解析やファイル読み込みを続けていないか。
- 生成コードの名前空間、accessibility、nullable annotation、`partial` 条件が利用側とずれていないか。
- hint name が一意で安定しており、同名型や nested type で衝突しないか。
- diagnostic は原因箇所に紐づき、ID、severity、message がレビューしやすいか。
- snapshot test や analyzer test で、生成結果と diagnostic の両方を確認しているか。

### 見落としやすい観点

- generator 内で現在時刻、乱数、絶対パス、環境変数を混ぜると、ビルドが非決定的になる。
- `context.AddSource("Generated.cs", sourceText)` のような固定名は、複数型や複数プロジェクトで衝突しやすい。
- syntax だけで拾うと、alias、partial、generic constraint、nullable context を見落としやすい。
- generated code の警告を呼び出し側に押しつけないよう、必要な `#nullable` や `GeneratedCode` 属性を整える。
- analyzer / generator のテストでは「生成されないべき入力」も確認する。

避けたい例:

```csharp
context.AddSource("Generated.cs", sourceText);
```

確認したい例:

```csharp
context.AddSource(
    $"{typeName}.Generated.g.cs",
    SourceText.From(source, Encoding.UTF8));
```

## unsafe / native interop

### レビューで見ること

- `unsafe` が必要な範囲を最小化し、通常の C# API との境界を薄くしているか。
- buffer length、終端文字、struct layout、packing、endianness、calling convention を明示しているか。
- 文字列 marshaling の encoding が native 側の期待と一致しているか。
- native resource の所有権を `SafeHandle` などで表現し、解放漏れと二重解放を避けているか。
- pointer、`Span<T>`、pinned object の lifetime が呼び出し中に限定されているか。
- platform、architecture、runtime version の差を起動時またはテストで検出できるか。
- P/Invoke の失敗時に `SetLastError = true` と `Marshal.GetLastPInvokeError()` などで原因を追えるか。
- native library の探索パスを暗黙にせず、配置、RID、fallback の方針があるか。

### 見落としやすい観点

- `IntPtr` は所有権が見えない。解放責務がある handle は `SafeHandle` に寄せる。
- `DllImport` の既定 charset や calling convention は、native 側と違ってもレビューで気づきにくい。
- struct の `bool`、`char`、配列、可変長 buffer はサイズがずれやすい。`Marshal.SizeOf` だけでなく native 定義と照合する。
- 32-bit / 64-bit で pointer size と alignment が変わる。CI の対象 architecture を確認する。
- `unsafe` の中身だけでなく、外側の public API が不正な length や null を渡せない形になっているかを見る。

避けたい例:

```csharp
[DllImport("native")]
private static extern IntPtr Open(string path);
```

確認したい例:

```csharp
[LibraryImport("native", EntryPoint = "open_file", SetLastError = true)]
private static partial SafeFileHandle OpenFile(
    [MarshalAs(UnmanagedType.LPUTF8Str)] string path);
```

## レビューで残す判断

- 何を守るためにこの仕組みを使っているか。
- 想定する最大サイズ、最大件数、タイムアウト、対応 OS はどこまでか。
- 失敗時にユーザー、運用者、開発者の誰がどう気づけるか。
- テストで代替実装に差し替えられる境界はどこか。
- 将来削除できる暫定対応なら、削除条件が書かれているか。

---

[その他の実務定石に戻る](README.md)
