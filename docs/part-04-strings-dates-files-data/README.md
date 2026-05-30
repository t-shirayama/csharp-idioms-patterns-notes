# Part 4: 文字列・日付・ファイル・データ形式

文字列、日付、ファイル、JSON、正規表現は、業務アプリケーションの境界で頻繁に現れます。
どれも一見単純に見えますが、culture、タイムゾーン、パス、文字コード、互換性など、実務では不具合の温床になりやすい領域です。

この Part では、C# の機能そのものよりも「どの前提を明示しておくか」を重視します。
レビューでは、正常系の短さだけでなく、入力の揺れ、環境差、将来の仕様変更に耐えられるかを確認します。

扱う対象はどれもアプリケーションの境界に近く、テスト環境では見えにくい差分を含みます。文字列は culture と入力の揺れ、日時はタイムゾーンと業務日付、ファイルは OS と権限、JSON は互換性、正規表現は複雑度とタイムアウトを意識します。

## 読み方

- 外部入力を受ける処理では、正常系より先に揺れと失敗の扱いを確認する。
- 保存形式や API 契約では、命名、日時、null、文字コードを明示する。
- OS、culture、タイムゾーンに依存する処理は、実行環境が変わっても同じ判断になるかを見る。
- 最適化 API や正規表現は、必要な箇所に絞り、テストで境界値を固定する。
- チェックリストでは、入力の正規化、保存形式、境界日時、パストラバーサル、JSON 互換性、Regex タイムアウトを優先して確認する。
- レビューコメントでは「この環境なら動く」ではなく、culture、OS、タイムゾーン、外部データの変更で壊れる条件を具体的に書く。

## 項目一覧

- [コードレビュー用チェックリスト](checklist.md)
- [string 比較、検索、分割、補間](string-比較-検索-分割-補間.md)
- [StringBuilder、Span / Memory の入口](stringbuilder-span-memory-の入口.md)
- [DateTime、DateOnly、TimeOnly、DateTimeOffset](datetime-dateonly-timeonly-datetimeoffset.md)
- [タイムゾーンと UTC](タイムゾーンと-utc.md)
- [file / path / directory 操作](file-path-directory-操作.md)
- [JSON serialization と System.Text.Json](json-serialization-と-system-text-json.md)
- [正規表現と GeneratedRegex](正規表現と-generatedregex.md)

---

[章別ノート一覧に戻る](../index.md)
