# C# Idioms & Practical Patterns Notes

実務で役立つ C# のイディオム、定石、設計パターン、落とし穴をまとめる日本語ノートです。

単なる文法メモではなく、「どの書き方を選ぶべきか」「どの場面で避けるべきか」「レビューで何を見るべきか」まで含めて整理します。

## 目的

- C# 8 以降の現代的な書き方を、実務判断に使える形で整理する
- nullable、record、pattern matching、async/await、LINQ などを安全に使う判断基準を増やす
- テストしやすさ、保守性、例外設計、パフォーマンス、レビュー観点をあわせて扱う
- コードレビュー時に確認しやすいチェックリストを提供する

## 対象

- C# の基本文法は分かるが、実務での書き方に迷う人
- .NET アプリケーションを保守・改善したい人
- コードレビューで使える判断基準を増やしたい人
- 設計、テスト、ログ、設定、非同期、パフォーマンスまで含めて C# コードを良くしたい人

## 読み方

- 公開サイトで読む: <https://t-shirayama.github.io/csharp-idioms-patterns-notes/>
- 全体像から探す: [docs/index.md](docs/index.md)
- すぐレビューに使う: 各 Part の `checklist.md`
- 個別トピックを深掘りする: 各 Part フォルダ内の項目別 Markdown

## Part 一覧

| Part | ノート | チェックリスト |
| --- | --- | --- |
| Part 1 | [現代 C# の基礎体力](docs/part-01-modern-csharp-basics/README.md) | [レビュー観点](docs/part-01-modern-csharp-basics/checklist.md) |
| Part 2 | [基本文法の実務イディオム](docs/part-02-basic-idioms/README.md) | [レビュー観点](docs/part-02-basic-idioms/checklist.md) |
| Part 3 | [C# 言語機能のステップアップ](docs/part-03-advanced-csharp-language/README.md) | [レビュー観点](docs/part-03-advanced-csharp-language/checklist.md) |
| Part 4 | [コレクションと LINQ](docs/part-04-collections-linq/README.md) | [レビュー観点](docs/part-04-collections-linq/checklist.md) |
| Part 5 | [文字列・日付・ファイル・データ形式](docs/part-05-strings-dates-files-data/README.md) | [レビュー観点](docs/part-05-strings-dates-files-data/checklist.md) |
| Part 6 | [オブジェクト指向と設計パターン](docs/part-06-oop-patterns/README.md) | [レビュー観点](docs/part-06-oop-patterns/checklist.md) |
| Part 7 | [非同期・並列・外部 I/O](docs/part-07-async-parallel-io/README.md) | [レビュー観点](docs/part-07-async-parallel-io/checklist.md) |
| Part 8 | [並行処理・バックグラウンド処理補足](docs/part-08-concurrency-background-services/README.md) | [レビュー観点](docs/part-08-concurrency-background-services/checklist.md) |
| Part 9 | [実務アプリケーションの定石](docs/part-09-application-practices/README.md) | [レビュー観点](docs/part-09-application-practices/checklist.md) |
| Part 10 | [ASP.NET Core / Web API 実務補足](docs/part-10-aspnetcore-webapi/README.md) | [レビュー観点](docs/part-10-aspnetcore-webapi/checklist.md) |
| Part 11 | [EF Core / DB アクセス深掘り](docs/part-11-efcore-data-access/README.md) | [レビュー観点](docs/part-11-efcore-data-access/checklist.md) |
| Part 12 | [テストしやすい C#](docs/part-12-testable-csharp/README.md) | [レビュー観点](docs/part-12-testable-csharp/checklist.md) |
| Part 13 | [スタイル、レビュー、保守性](docs/part-13-style-review-maintainability/README.md) | [レビュー観点](docs/part-13-style-review-maintainability/checklist.md) |
| Part 14 | [パフォーマンスと落とし穴](docs/part-14-performance-pitfalls/README.md) | [レビュー観点](docs/part-14-performance-pitfalls/checklist.md) |
| Part 15 | [運用・品質・配布補足](docs/part-15-operations-quality-delivery/README.md) | [レビュー観点](docs/part-15-operations-quality-delivery/checklist.md) |
| Appendix | [その他の実務定石](docs/appendix/README.md) | [レビュー観点](docs/appendix/checklist.md) |

## 各項目で扱う観点

- 使いどころ
- 判断基準
- 良い例
- 避けたい例
- 改善例
- テストしやすさ
- パフォーマンス上の注意
- コードレビュー観点

## 執筆方針

- C# 8 以降の現代的な書き方を中心にする
- 古い書き方も、現場で遭遇しやすいものは比較対象として扱う
- サンプルコードは短く、意図が分かる単位にする
- 仕様説明で終わらせず、実務での判断基準を書く
- パフォーマンスの話は、必要な場面と過剰最適化の境界も書く
