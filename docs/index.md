# C# Idioms & Practical Patterns Notes

実務で役立つ C# のイディオム、定石、設計パターン、落とし穴をまとめる日本語ノートです。

## 読み方

- 全体像から探す: このページの Part 一覧
- すぐレビューに使う: 各 Part のチェックリスト
- 個別トピックを深掘りする: 各 Part フォルダ内の項目別ページ

## Part 一覧

| Part | ノート | チェックリスト |
| --- | --- | --- |
| Part 1 | [現代 C# の基礎体力](part-01-modern-csharp-basics/README.md) | [レビュー観点](part-01-modern-csharp-basics/checklist.md) |
| Part 2 | [基本文法の実務イディオム](part-02-basic-idioms/README.md) | [レビュー観点](part-02-basic-idioms/checklist.md) |
| Part 3 | [C# 言語機能のステップアップ](part-03-advanced-csharp-language/README.md) | [レビュー観点](part-03-advanced-csharp-language/checklist.md) |
| Part 4 | [コレクションと LINQ](part-04-collections-linq/README.md) | [レビュー観点](part-04-collections-linq/checklist.md) |
| Part 5 | [文字列・日付・ファイル・データ形式](part-05-strings-dates-files-data/README.md) | [レビュー観点](part-05-strings-dates-files-data/checklist.md) |
| Part 6 | [オブジェクト指向と設計パターン](part-06-oop-patterns/README.md) | [レビュー観点](part-06-oop-patterns/checklist.md) |
| Part 7 | [非同期・並列・外部 I/O](part-07-async-parallel-io/README.md) | [レビュー観点](part-07-async-parallel-io/checklist.md) |
| Part 8 | [並行処理・バックグラウンド処理補足](part-08-concurrency-background-services/README.md) | [レビュー観点](part-08-concurrency-background-services/checklist.md) |
| Part 9 | [実務アプリケーションの定石](part-09-application-practices/README.md) | [レビュー観点](part-09-application-practices/checklist.md) |
| Part 10 | [ASP.NET Core / Web API 実務補足](part-10-aspnetcore-webapi/README.md) | [レビュー観点](part-10-aspnetcore-webapi/checklist.md) |
| Part 11 | [EF Core / DB アクセス深掘り](part-11-efcore-data-access/README.md) | [レビュー観点](part-11-efcore-data-access/checklist.md) |
| Part 12 | [テストしやすい C#](part-12-testable-csharp/README.md) | [レビュー観点](part-12-testable-csharp/checklist.md) |
| Part 13 | [スタイル、レビュー、保守性](part-13-style-review-maintainability/README.md) | [レビュー観点](part-13-style-review-maintainability/checklist.md) |
| Part 14 | [パフォーマンスと落とし穴](part-14-performance-pitfalls/README.md) | [レビュー観点](part-14-performance-pitfalls/checklist.md) |
| Part 15 | [運用・品質・配布補足](part-15-operations-quality-delivery/README.md) | [レビュー観点](part-15-operations-quality-delivery/checklist.md) |
| Appendix | [その他の実務定石](appendix/README.md) | [レビュー観点](appendix/checklist.md) |

<!-- TODO対応追記 -->

## TODO対応: Part 一覧の下に「目的別索引」を追加し

> 対応元: P1 / Part 一覧の下に「目的別索引」を追加し、外部コマンド、環境設定、配布物、メタプログラミング、native 境界、レビュー用チェックリストへ直接リンクする。

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

## 目的別索引

| 目的 | 入口 | 関連チェック |
| --- | --- | --- |
| 外部コマンドを安全に起動したい | [プロセス起動](appendix/プロセス起動.md) | [Appendix レビュー観点](appendix/checklist.md) |
| 一時ファイル、ZIP、作業ディレクトリを安全に扱いたい | [一時ファイルと作業ディレクトリ](appendix/一時ファイルと作業ディレクトリ.md)、[ZIP 操作](appendix/zip-操作.md) | [文字列・日付・ファイル レビュー観点](part-05-strings-dates-files-data/checklist.md) |
| 環境設定や secret の運用を整理したい | [環境変数](appendix/環境変数.md)、[configuration と secret operations](part-15-operations-quality-delivery/configuration-and-secret-operations.md) | [運用レビュー観点](part-15-operations-quality-delivery/checklist.md) |
| NuGet、バージョン、配布物を確認したい | [バージョン情報](appendix/バージョン情報.md)、[NuGet / versioning / library design](part-15-operations-quality-delivery/nuget-versioning-library-design.md) | [運用レビュー観点](part-15-operations-quality-delivery/checklist.md) |
| reflection、source generator、NativeAOT を判断したい | [reflection](appendix/reflection.md)、[source generator の入口](appendix/source-generator-の入口.md)、[trimming / NativeAOT 互換性](appendix/trimming-nativeaot-互換性.md) | [Appendix レビュー観点](appendix/checklist.md) |
| native 境界や unsafe を扱いたい | [unsafe / native interop の入口](appendix/unsafe-native-interop-の入口.md) | [Appendix レビュー観点](appendix/checklist.md) |
| レビュー用チェックリストを横断したい | [全チェックリスト導線](checklists.md) | 各 Part の `checklist.md` |

