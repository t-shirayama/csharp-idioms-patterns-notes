# C# Idioms & Practical Patterns Notes: 章別ノート

README.md の総合目次を、章別に読み進めるためのノート置き場です。

各 Part では、単なる文法説明ではなく、実務での使いどころ、避けたい書き方、テストしやすさ、レビュー観点を扱います。

## Part 別ノート

| Part | ノート | チェックリスト | 扱う範囲 |
| --- | --- | --- | --- |
| Part 1 | [現代 C# の基礎体力](part-01-modern-csharp-basics/README.md) | [レビュー観点](part-01-modern-csharp-basics/checklist.md) | 基礎体力 |
| Part 2 | [基本文法の実務イディオム](part-02-basic-idioms/README.md) | [レビュー観点](part-02-basic-idioms/checklist.md) | 基本イディオム |
| Part 3 | [コレクションと LINQ](part-03-collections-linq/README.md) | [レビュー観点](part-03-collections-linq/checklist.md) | コレクションと LINQ |
| Part 4 | [文字列・日付・ファイル・データ形式](part-04-strings-dates-files-data/README.md) | [レビュー観点](part-04-strings-dates-files-data/checklist.md) | 文字列・日付・ファイル・データ形式 |
| Part 5 | [オブジェクト指向と設計パターン](part-05-oop-patterns/README.md) | [レビュー観点](part-05-oop-patterns/checklist.md) | オブジェクト指向と設計パターン |
| Part 6 | [非同期・並列・外部 I/O](part-06-async-parallel-io/README.md) | [レビュー観点](part-06-async-parallel-io/checklist.md) | 非同期・並列・外部 I/O |
| Part 7 | [実務アプリケーションの定石](part-07-application-practices/README.md) | [レビュー観点](part-07-application-practices/checklist.md) | 実務アプリケーション |
| Part 8 | [テストしやすい C#](part-08-testable-csharp/README.md) | [レビュー観点](part-08-testable-csharp/checklist.md) | テスト設計 |
| Part 9 | [スタイル、レビュー、保守性](part-09-style-review-maintainability/README.md) | [レビュー観点](part-09-style-review-maintainability/checklist.md) | スタイル・レビュー・保守性 |
| Part 10 | [パフォーマンスと落とし穴](part-10-performance-pitfalls/README.md) | [レビュー観点](part-10-performance-pitfalls/checklist.md) | パフォーマンスと落とし穴 |
| Appendix | [その他の実務定石](appendix/README.md) | [レビュー観点](appendix/checklist.md) | 補足項目 |

## 読み方

- 迷ったときは README.md の総合目次から全体像を確認する。
- 実装中に困ったときは、該当する Part の「避けたい書き方」と「レビュー観点」を先に見る。
- サンプルコードはそのまま貼るよりも、プロジェクトの設計や命名に合わせて調整する。
- 迷ったら「良い例」より先に「避けたい例」と「レビュー観点」を読む。

## 追記するときの型

各項目に追記するときは、できるだけ次の型に合わせます。

1. まず「なぜその書き方を選ぶのか」を説明する
2. 短い `csharp` コード例を置く
3. 避けたい書き方を対比する
4. レビューで確認する観点を書く

## 品質の目安

- 読者が「いつ使うか」と「いつ避けるか」を判断できる。
- コード例が短く、レビュー時に探すべき匂いが分かる。
- パフォーマンスやテスト容易性の話は、必要な場面と過剰な場面を分ける。

