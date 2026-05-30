# Part 4 コードレビュー用チェックリスト

文字列、日時、ファイル、JSON、正規表現は、外部入力や実行環境の差が不具合として現れやすい領域です。
レビューでは、正常系の短さだけでなく、culture、タイムゾーン、文字コード、パス、互換性、失敗時の扱いを確認します。

## 文字列

- [ ] ID、コード、キー、ファイル拡張子、HTTP ヘッダー名などの機械的な文字列比較で、`StringComparison.Ordinal` / `OrdinalIgnoreCase` を指定している。
- [ ] `ToLower()` や `ToUpper()` で正規化してから比較していない。必要なら comparer や `StringComparison` を使っている。
- [ ] `Dictionary<string, ...>` や `HashSet<string>` で、キーの大小文字、culture、前後空白の扱いを comparer と入力正規化で明示している。
- [ ] ユーザー向けの検索、表示名、並び替えでは、対象 culture と仕様を確認している。
- [ ] null、空文字、空白のみの文字列を同一視するか区別するかが明確である。
- [ ] 入力の `Trim`、Unicode 正規化、全角半角、改行混入をどこで扱うかが決まっている。
- [ ] `Split` の結果に空要素、前後空白、想定外の列数が混じる場合の扱いが決まっている。
- [ ] CSV、JSON、HTML、ログ形式など構造を持つ入力を、単純な `Split` や手書き処理で抱え込んでいない。
- [ ] ログ、例外メッセージ、監査情報に個人情報、secret、トークン、パスワードを埋め込んでいない。
- [ ] 外部連携や保存形式に出す日時、数値、通貨を、現在 culture に依存する文字列補間で出力していない。
- [ ] 外部向け文字列化には `CultureInfo.InvariantCulture` や明示 culture を使い、表示用の culture と保存用の形式を混ぜていない。

## 文字列生成と最適化

- [ ] 数個の値を組み立てるだけの処理を、過剰に `StringBuilder` へ置き換えていない。
- [ ] ループで多数の行や断片を組み立てる処理では、`StringBuilder` や `string.Join` を検討している。
- [ ] hot path の substring 多用を最適化する場合、`ReadOnlySpan<char>` で読み取るだけにできるか確認している。
- [ ] `Span<T>` を public API や複雑な境界へ広げる前に、読みやすさと利用者側の制約を確認している。
- [ ] async 境界や保持が必要なバッファに `Span<T>` を渡そうとしていない。必要なら `Memory<T>` を使っている。
- [ ] allocation 削減の変更で、入力検証、culture、文字コード、例外処理の意味が変わっていない。
- [ ] 最適化の根拠が、計測結果、呼び出し頻度、データ量のいずれかに結びついている。

## 日付と時刻

- [ ] 誕生日、営業日、請求対象日など日付だけの値に `DateOnly` を使う選択肢を検討している。
- [ ] 開店時刻、締切時刻など日付を持たない値に `TimeOnly` を使う選択肢を検討している。
- [ ] 外部 API、DB、ログなど絶対時刻を扱う境界では、`DateTimeOffset` を優先している。
- [ ] `DateTime.Kind` が `Unspecified` の値を、暗黙に UTC や local として扱っていない。
- [ ] `DateTimeOffset` が offset を持つ値であり、タイムゾーン名や夏時間ルールまでは保持しないことを前提にしている。
- [ ] `DateTime.Now`、`DateTime.UtcNow`、`DateTimeOffset.UtcNow` が業務ロジックに散らばらず、`TimeProvider` などで注入できる。
- [ ] DB や JSON に保存する日時形式、offset の有無、精度、タイムゾーン変換位置が明確である。
- [ ] 日付だけの仕様を 0 時 0 分の `DateTime` で代用して、境界日のバグを作っていない。
- [ ] 期間条件は `開始 <= x < 終了` のような半開区間で表せるか確認し、月末、日末、ミリ秒丸めに依存していない。
- [ ] 外部入力の日時 parse では、`Parse` 任せにせず、許可する形式、culture、失敗時の扱いを明示している。
- [ ] テストで現在時刻を固定でき、月末、年末、うるう日、日付変更直後のケースを確認できる。

## タイムゾーン

- [ ] 作成日時、更新日時、イベント発生時刻などの保存形式は UTC を基本にしている。
- [ ] 表示や帳票出力では、ユーザー、店舗、契約、業務拠点など、どのタイムゾーンを使うか明示している。
- [ ] 「現地日付」で集計する仕様を、UTC 日付で丸めて処理していない。
- [ ] 現地日付の範囲検索では、現地の開始・終了を UTC に変換してから検索している。
- [ ] サーバーの `DateTime.Today`、`TimeZoneInfo.Local`、OS のローカル設定に業務仕様が依存していない。
- [ ] タイムゾーン ID をコードに散らさず、設定値やユーザープロファイルとして扱っている。
- [ ] Windows と Linux の両方で動くサービスでは、タイムゾーン ID の互換性を確認している。
- [ ] `TimeZoneInfo.FindSystemTimeZoneById` の失敗、設定ミス、廃止された ID を例外として扱える。
- [ ] 夏時間の開始・終了、存在しない時刻、曖昧な時刻の扱いが仕様とテストで確認できる。

## ファイルとパス

- [ ] パス結合に文字列連結ではなく、`Path.Combine`、`Path.Join`、`Path.GetFullPath` などを使っている。
- [ ] ユーザー入力を含むパスが、許可した base directory の内側に収まるか検証している。
- [ ] base directory 検証では単純な `StartsWith` だけに頼らず、正規化後の区切り文字、大文字小文字、シンボリックリンクを考慮している。
- [ ] 拡張子、ファイル名、上書き可否、サイズ上限、保存先の権限を確認している。
- [ ] 拡張子だけでファイル種別を信頼せず、必要に応じて MIME type や内容の先頭 bytes も確認している。
- [ ] ファイルが存在しない、権限がない、別プロセスが使用中、ディスク不足といった I/O 失敗を想定している。
- [ ] `File.Exists` の結果を過信せず、実際の読み書き時の例外を扱っている。
- [ ] ASP.NET Core などのリクエスト処理で、不要な同期 I/O によってスレッドを塞いでいない。
- [ ] 大きなファイルや逐次処理では、全体を `ReadAllText` / `ReadAllBytes` で読み込まず `Stream` を検討している。
- [ ] 文字コード、BOM の有無、改行コードの前提が外部仕様として明示されている。
- [ ] 長時間の I/O や非同期処理に `CancellationToken` を渡している。
- [ ] 一時ファイルやアップロード保存では、ファイル名衝突、部分書き込み、後始末、原子的な置き換えを確認している。
- [ ] `Stream`、`FileStream`、`ZipArchive` など disposable なリソースを `using` / `await using` で閉じている。

## JSON とデータ形式

- [ ] 外部公開 API や保存形式で、domain model を直接 serialize せず DTO を用意している。
- [ ] `JsonSerializerOptions` が呼び出し箇所ごとに分散せず、命名規則、null、enum、converter の方針が統一されている。
- [ ] `required`、nullable、既定値、欠落したプロパティの扱いが、deserialize 後のコードで明確である。
- [ ] `PropertyNameCaseInsensitive`、unknown property、コメント、末尾カンマを許すかどうかを外部契約として決めている。
- [ ] enum を数値で出すか文字列で出すかを、互換性と可読性のトレードオフとして決めている。
- [ ] 日時、decimal、ID の JSON 表現が外部仕様として説明できる。
- [ ] プロパティ名変更、削除、型変更、enum 値追加、null 省略変更が破壊的変更にならないか確認している。
- [ ] 追加フィールドを保持して再出力する必要がある場合、`JsonExtensionData` などの方針を検討している。
- [ ] custom converter には必要な理由があり、正常系だけでなく不正入力のテストがある。
- [ ] deserialize 失敗を握りつぶして、空 DTO や既定値で処理を続けていない。
- [ ] 大きな JSON やストリーミング API では、全体を文字列化せず `Stream` / `Utf8JsonReader` / source generation を検討している。
- [ ] 循環参照や polymorphism を serializer option で安易に通さず、公開契約として安全か確認している。

## 正規表現

- [ ] `StartsWith`、`EndsWith`、`Contains`、`IndexOf` で足りる処理を、正規表現で複雑にしていない。
- [ ] 同じ pattern を複数箇所に重複して書かず、`GeneratedRegex` などで名前を付けている。
- [ ] 外部入力に適用する正規表現では、タイムアウトや backtracking のリスクを確認している。
- [ ] nested quantifier、曖昧な alternation、任意長の `.*` が長い入力で極端に遅くならないか確認している。
- [ ] 機械的な形式の大小文字無視では、culture の影響を避けるため `CultureInvariant` などの options を検討している。
- [ ] `^` / `$`、`.`、改行、複数行 option の意味が、期待する入力全体の一致条件に合っている。
- [ ] 入力全体に一致させたい場合、必要に応じて `\A` / `\z` や `RegexOptions.Singleline` / `Multiline` の違いを確認している。
- [ ] ユーザー入力を pattern に埋め込む場合、`Regex.Escape` とタイムアウトを使っている。
- [ ] 正規表現を形式チェックまでに留め、重複確認、DB 照合、権限判定などの業務ルールを混ぜていない。
- [ ] `RegexOptions.Compiled` や `GeneratedRegex` を使う理由があり、起動時間、AOT、呼び出し頻度とのトレードオフを確認している。
- [ ] 成功例だけでなく、境界値、失敗例、大小文字、culture、長い入力のテストがある。

---

[Part 4 README に戻る](README.md)
