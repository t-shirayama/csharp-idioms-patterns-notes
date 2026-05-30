# Part 5: 文字列・日付・ファイル・データ形式

文字列、日付、ファイル、JSON、正規表現は、業務アプリケーションの境界で頻繁に現れます。
入力の揺れ、culture、タイムゾーン、パス、文字コード、互換性、正規表現の計算量は、単体テストだけでは見逃しやすい不具合につながります。

この Part では、C# の API を暗記するよりも、「どの前提を明示するか」「どこで正規化するか」「保存形式と表示形式を分けているか」を重視します。
レビューでは、手元の環境で動くかだけでなく、OS、地域設定、時刻、外部データ、将来のスキーマ変更で壊れないかを確認します。

## このPartで身につくこと

- `string` の比較、検索、分割、補間で、culture と ordinal 比較を使い分ける。
- `StringBuilder`、`Span<T>`、`Memory<T>` を、必要な場面に限定して検討する。
- `DateTime`、`DateOnly`、`TimeOnly`、`DateTimeOffset` を、業務日付、時刻、絶対時刻の違いで選ぶ。
- UTC、ローカル時刻、タイムゾーン、表示形式、保存形式を分けて扱う。
- file、path、directory 操作で、OS 差、権限、相対パス、パストラバーサルを意識する。
- `System.Text.Json` と正規表現を、互換性、性能、保守性の観点で使い分ける。

## 読む順番

1. まず [string 比較、検索、分割、補間](string-比較-検索-分割-補間.md) で、文字列処理の既定値に頼らない判断を押さえる。
2. 次に [DateTime、DateOnly、TimeOnly、DateTimeOffset](datetime-dateonly-timeonly-datetimeoffset.md) と [タイムゾーンと UTC](タイムゾーンと-utc.md) で、日時の意味を整理する。
3. [file / path / directory 操作](file-path-directory-操作.md) で、ファイルシステム境界の失敗条件を確認する。
4. [JSON serialization と System.Text.Json](json-serialization-と-system-text-json.md) で、API 契約と互換性を見る。
5. [正規表現と GeneratedRegex](正規表現と-generatedregex.md) と [StringBuilder、Span / Memory の入口](stringbuilder-span-memory-の入口.md) は、必要な場面で性能と保守性の観点から読む。
6. 最後に [コードレビュー用チェックリスト](checklist.md) で、境界処理のレビュー観点に落とし込む。

## 重要トピック

- 文字列比較は、識別子やプロトコル値なら `StringComparison.Ordinal` 系、表示や自然言語なら culture を検討する。
- 日時は「日付だけ」「時刻だけ」「タイムゾーン付きの瞬間」「ローカル表示」を混ぜない。
- ファイルパスは文字列連結ではなく `Path` API で扱い、ユーザー入力をそのまま結合しない。
- JSON は C# の都合ではなく、外部契約としてプロパティ名、null、既定値、日時形式、互換性を決める。
- 正規表現は便利だが、複雑な仕様を隠しやすい。タイムアウト、可読性、テストケースをセットで扱う。

## 実務での使いどころ

- API 入力、CSV、ログ、設定ファイルなど、外部から来る文字列を境界で正規化する。
- 業務日、締め時刻、予約時刻、監査ログ時刻のように、日時の意味が違う値を型と命名で分ける。
- ファイルアップロード、バッチ処理、エクスポート機能で、パス、権限、存在確認、競合、文字コードを扱う。
- Web API やメッセージ連携で、`System.Text.Json` のオプションを契約として固定する。
- 複雑な入力検証や抽出で正規表現を使う場合、仕様変更に耐える名前付けとテストを用意する。

## よくある落とし穴

- `ToLower()` や `ToUpper()` を使って比較し、culture や余計な allocation の問題を招く。
- `DateTime.Now` と `DateTime.UtcNow`、`DateTimeKind`、`DateTimeOffset` の意味を混ぜる。
- 日付だけの業務値を `DateTime` の午前0時として扱い、タイムゾーン変換で日付がずれる。
- パスを文字列連結し、OS 差や `..` を含む入力に弱くなる。
- JSON のプロパティ名や null の扱いを暗黙の既定値に任せ、クライアント互換性を壊す。
- 正規表現にタイムアウトを設定せず、悪意ある入力や巨大入力で処理が詰まる。
- `Span<T>` や `Memory<T>` を必要以上に使い、単純なコードを読みにくくする。

## レビュー観点

- 文字列比較に `StringComparison` が明示されているか。識別子と表示文字列を同じ比較にしていないか。
- 日時の型、保存形式、表示形式、タイムゾーンの責務が分かれているか。
- ファイルパスは `Path.Combine`、`Path.GetFullPath` などで安全に扱われているか。
- 外部入力の検証、正規化、失敗時の扱いが境界で閉じているか。
- JSON のオプション、プロパティ名、null、日時、未知プロパティへの方針が契約として明確か。
- 正規表現は読みやすく、タイムアウトと境界値テストがあるか。`GeneratedRegex` を使う理由は明確か。

## 記事一覧

- [コードレビュー用チェックリスト](checklist.md): 文字列、日時、I/O、JSON、Regex の境界リスクをレビューで確認する。
- [string 比較、検索、分割、補間](string-比較-検索-分割-補間.md): culture、ordinal、検索、分割、補間を安全に使い分ける。
- [StringBuilder、Span / Memory の入口](stringbuilder-span-memory-の入口.md): 文字列処理の allocation と低レベル API を、必要な場面に絞って検討する。
- [DateTime、DateOnly、TimeOnly、DateTimeOffset](datetime-dateonly-timeonly-datetimeoffset.md): 日付、時刻、瞬間、オフセット付き時刻を型で表現する。
- [タイムゾーンと UTC](タイムゾーンと-utc.md): 保存、計算、表示で UTC とタイムゾーンをどう扱うか整理する。
- [file / path / directory 操作](file-path-directory-操作.md): ファイル、パス、ディレクトリ操作の OS 差、権限、入力検証を扱う。
- [JSON serialization と System.Text.Json](json-serialization-と-system-text-json.md): JSON 契約、オプション、互換性、日時、null の扱いを整理する。
- [正規表現と GeneratedRegex](正規表現と-generatedregex.md): Regex の判定、抽出、置換、性能、タイムアウト、`GeneratedRegex` を扱う。

---

[章別ノート一覧に戻る](../index.md)
