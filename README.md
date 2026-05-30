# C# Idioms & Practical Patterns Notes

実務で役立つ C# のイディオム、定石、設計パターン、落とし穴をまとめるリポジトリです。

単なる文法メモではなく、「どの書き方を選ぶべきか」「どの場面で避けるべきか」「レビューで何を見るべきか」まで含めて整理します。

## 目的

次のような判断に使える知識を、C# 8 以降の現代的な書き方を中心にまとめます。

- どの書き方が読みやすいか
- どのパターンが保守しやすいか
- どの場面で避けるべきか
- テストしやすいコードにするにはどう書くか
- パフォーマンスや例外処理で注意すべき点は何か
- コードレビューでどの観点を見るべきか
- .NET アプリケーションとして運用しやすい設計にするにはどうするか

## 対象

- C# の基本文法は分かるが、実務での書き方に迷う人
- .NET アプリケーションを保守・改善したい人
- コードレビューで使える判断基準を増やしたい人
- nullable、record、pattern matching、async/await、LINQ などを実務で使いこなしたい人
- テスト、ログ、設定、例外設計、パフォーマンスまで含めて C# コードを良くしたい人

## このリポジトリで扱う範囲

各トピックでは、できるだけ次の観点をセットで扱います。

- 使いどころ
- 基本形
- 実務でよく使う書き方
- 避けたい書き方
- テストしやすさ
- パフォーマンス上の注意
- コードレビュー観点

## 章別ノート

各 Part の本文は [docs/README.md](docs/README.md) から参照できます。

| Part | 章別ノート | 範囲 |
| --- | --- | --- |
| Part 1 | [現代 C# の基礎体力](docs/part-01-modern-csharp-basics/README.md) | 基礎体力 |
| Part 2 | [基本文法の実務イディオム](docs/part-02-basic-idioms/README.md) | 基本イディオム |
| Part 3 | [コレクションと LINQ](docs/part-03-collections-linq/README.md) | コレクションと LINQ |
| Part 4 | [文字列・日付・ファイル・データ形式](docs/part-04-strings-dates-files-data/README.md) | 文字列・日付・ファイル・データ形式 |
| Part 5 | [オブジェクト指向と設計パターン](docs/part-05-oop-patterns/README.md) | オブジェクト指向と設計パターン |
| Part 6 | [非同期・並列・外部 I/O](docs/part-06-async-parallel-io/README.md) | 非同期・並列・外部 I/O |
| Part 7 | [実務アプリケーションの定石](docs/part-07-application-practices/README.md) | 実務アプリケーション |
| Part 8 | [テストしやすい C#](docs/part-08-testable-csharp/README.md) | テスト設計 |
| Part 9 | [スタイル、レビュー、保守性](docs/part-09-style-review-maintainability/README.md) | スタイル・レビュー・保守性 |
| Part 10 | [パフォーマンスと落とし穴](docs/part-10-performance-pitfalls/README.md) | パフォーマンスと落とし穴 |
| Appendix | [その他の実務定石](docs/appendix/README.md) | 補足項目 |

## 目次

### Part 1: 現代 C# の基礎体力

#### C# と .NET の全体像

- C# と .NET の関係
- .NET Runtime、SDK、Target Framework
- Console、Web API、Worker Service などの代表的なアプリケーション
- 言語機能とランタイム機能の違い

#### 型、値型、参照型、nullable reference types

- 値型と参照型
- nullable value types
- nullable reference types
- null 許容設計の基本
- null 警告との付き合い方

#### class / struct / record / record struct

- class を選ぶ場面
- struct を選ぶ場面
- record を選ぶ場面
- record struct を選ぶ場面
- DTO、Value Object、Entity との関係

#### property、init、required、readonly

- 自動実装プロパティ
- get-only property
- init-only property
- required member
- readonly field
- immutable に近づける設計

#### namespace、using、file-scoped namespace

- namespace の切り方
- using directive
- global using
- file-scoped namespace
- プロジェクト構造と名前空間の対応

#### exception、using、IDisposable、リソース管理

- 例外の基本
- using statement と using declaration
- IDisposable / IAsyncDisposable
- ファイル、ストリーム、DB 接続の解放
- リソースリークを防ぐレビュー観点

### Part 2: 基本文法の実務イディオム

#### 初期化、代入、ガード節

- オブジェクト初期化子
- コレクション初期化子
- collection expression
- ガード節
- 早期 return
- 引数検証

#### if / switch / pattern matching

- if を読みやすく書く
- switch expression
- type pattern
- property pattern
- relational pattern
- list pattern
- 複雑な条件式を分解する判断基準

#### null 処理

- null 合体演算子
- null 条件演算子
- null 合体代入
- ArgumentNullException.ThrowIfNull
- null を返す設計と返さない設計
- Optional 的な表現の検討

#### enum、flags enum、代替設計

- enum の基本
- flags enum
- enum と switch
- enum の保存形式
- enum ではなく class や record を使う場面

#### range、index、tuple、deconstruction、匿名型

- index と range
- tuple の使いどころ
- deconstruction
- 匿名型
- 戻り値設計としての tuple の限界

### Part 3: コレクションと LINQ

#### array、List、Dictionary、HashSet

- array と List の使い分け
- Dictionary の基本
- HashSet の基本
- Queue、Stack
- コレクション選択の判断基準

#### IEnumerable / ICollection / IReadOnlyCollection の使い分け

- IEnumerable を公開する場面
- ICollection を受け取る場面
- IReadOnlyCollection / IReadOnlyList
- 遅延評価を API 境界に持ち込むリスク

#### LINQ の基本、遅延評価、即時評価

- Select
- Where
- Any / All
- First / FirstOrDefault / Single / SingleOrDefault
- ToList / ToArray
- 遅延評価と副作用

#### LINQ を使いこなす

- GroupBy
- Join
- SelectMany
- Aggregate
- OrderBy / ThenBy
- DistinctBy / MinBy / MaxBy
- query syntax と method syntax

#### LINQ で避けたい書き方とパフォーマンス

- 多重列挙
- Count() と Any() の使い分け
- LINQ の過剰なネスト
- 大量データでのメモリ使用
- DB クエリに変換される LINQ の注意点

### Part 4: 文字列・日付・ファイル・データ形式

#### string 比較、検索、分割、補間

- string.Equals
- StringComparison
- Contains / StartsWith / EndsWith
- Split / Join
- string interpolation
- culture を意識した文字列処理

#### StringBuilder、Span / Memory の入口

- StringBuilder を使う場面
- 文字列連結で十分な場面
- ReadOnlySpan<char>
- Memory<T>
- allocation を減らしたい場面の考え方

#### DateTime、DateOnly、TimeOnly、DateTimeOffset

- DateTime
- DateOnly
- TimeOnly
- DateTimeOffset
- 現在時刻の扱い
- テストしやすい時刻設計

#### タイムゾーンと UTC

- UTC 保存の基本
- ローカル時刻への変換
- TimeZoneInfo
- 日付だけを扱う場合の注意
- タイムゾーン起因のバグ

#### file / path / directory 操作

- File
- Directory
- Path
- Stream
- 非同期ファイル I/O
- 一時ファイル
- パス結合の落とし穴

#### JSON serialization と System.Text.Json

- JsonSerializer
- naming policy
- null の扱い
- enum の扱い
- custom converter
- DTO 設計
- バージョン互換性

#### 正規表現と GeneratedRegex

- Regex の基本
- Match / Matches
- Replace / Split
- RegexOptions
- GeneratedRegex
- 正規表現を使いすぎない判断

### Part 5: オブジェクト指向と設計パターン

#### カプセル化、継承、ポリモーフィズム

- カプセル化
- 継承
- ポリモーフィズム
- virtual / override
- sealed
- 継承より合成を選ぶ場面

#### interface と abstract class の使い分け

- interface を選ぶ場面
- abstract class を選ぶ場面
- default interface methods
- テストダブルと interface
- API として公開する場合の互換性

#### Template Method、Strategy、Factory

- Template Method パターン
- Strategy パターン
- Factory Method
- Abstract Factory
- パターンを入れすぎない判断

#### Dependency Injection

- DI の基本
- constructor injection
- service lifetime
- options injection
- DI コンテナに依存しすぎない設計
- テストしやすさとの関係

#### Value Object、Entity、DTO

- Value Object
- Entity
- DTO
- record と DTO
- 同一性と等価性
- 層をまたぐモデルの分離

#### SOLID 原則を C# コードで読む

- Single Responsibility Principle
- Open-Closed Principle
- Liskov Substitution Principle
- Interface Segregation Principle
- Dependency Inversion Principle
- 原則を機械的に当てはめない判断

### Part 6: 非同期・並列・外部 I/O

#### async / await の基本

- async method
- Task
- await
- async void を避ける
- 例外伝播
- 非同期メソッドの命名

#### Task、ValueTask、CancellationToken

- Task と Task<T>
- ValueTask
- CancellationToken
- キャンセル可能な API 設計
- タイムアウトとの違い

#### ConfigureAwait の現代的な扱い

- ConfigureAwait(false)
- アプリケーション種別による違い
- ライブラリコードでの考え方
- 古い知識との整理

#### HttpClient の定石

- HttpClient の再利用
- IHttpClientFactory
- timeout
- retry
- cancellation
- JSON API 呼び出し
- エラーレスポンスの扱い

#### 並列処理、Parallel、PLINQ、Channel

- 並行と並列の違い
- Parallel.ForEach
- PLINQ
- Channel
- lock
- thread-safe collection
- CPU-bound と I/O-bound

#### タイムアウト、リトライ、キャンセル設計

- タイムアウトの責務
- リトライしてよい処理
- リトライしてはいけない処理
- CancellationToken の伝播
- 外部サービス障害への備え

### Part 7: 実務アプリケーションの定石

#### 設定と Options pattern

- appsettings.json
- environment variables
- Options pattern
- IOptions / IOptionsSnapshot / IOptionsMonitor
- secret を設定に置かない設計

#### ログと構造化ログ

- ILogger
- log level
- structured logging
- exception logging
- 個人情報をログに出さない
- 調査しやすいログの粒度

#### validation

- 入力検証
- domain validation
- DataAnnotations
- FluentValidation
- 例外と validation error の使い分け

#### エラー処理と例外設計

- 例外を投げる場面
- Result 型的な設計
- 例外の握りつぶしを避ける
- 独自例外
- エラーメッセージ
- 監視しやすい失敗設計

#### repository / service / usecase 分離

- repository
- application service
- domain service
- usecase
- controller を薄く保つ
- 層の責務を混ぜない

#### Web API での DTO、境界、マッピング

- request DTO
- response DTO
- domain model を直接返さない判断
- mapping
- API バージョニング
- backward compatibility

#### DB アクセス時の注意点

- connection
- transaction
- N+1
- lazy loading
- migration
- concurrency
- repository の抽象化しすぎ問題

### Part 8: テストしやすい C#

#### 単体テストしやすい設計

- pure function
- 副作用の分離
- constructor injection
- static 依存を減らす
- テストしやすさと設計の関係

#### xUnit / NUnit / MSTest の観点

- テストフレームワークの違い
- arrange / act / assert
- parameterized test
- setup / teardown
- fixture

#### mock / stub / fake の使い分け

- mock
- stub
- fake
- spy
- mock しすぎるテストの問題
- 外部 I/O の置き換え

#### 時刻、乱数、外部 I/O の分離

- TimeProvider
- Random の扱い
- ファイル I/O の分離
- HTTP 呼び出しの分離
- DB の分離
- deterministic なテスト

#### テストデータビルダー

- object mother
- test data builder
- fixture builder
- 変更に強いテストデータ
- 読みやすいテスト名

#### 例外・非同期・境界値のテスト

- 例外のテスト
- async method のテスト
- cancellation のテスト
- boundary value
- null / empty / invalid input

### Part 9: スタイル、レビュー、保守性

#### 命名規則

- class naming
- method naming
- variable naming
- boolean naming
- async method naming
- 略語の扱い

#### コメントと XML documentation

- コメントを書く場面
- コメントを書かずにコードで表す場面
- XML documentation
- public API の説明
- 古いコメントを残さない

#### メソッド分割、責務分割

- 長すぎるメソッド
- 引数が多すぎるメソッド
- private method の切り出し
- class の責務
- cohesion と coupling

#### 読みやすい条件式

- 条件式の分解
- boolean variable
- guard clause
- negation を減らす
- 複雑な switch の扱い

#### コードレビュー観点

- correctness
- readability
- maintainability
- testability
- performance
- security
- operational readiness

#### analyzer、EditorConfig、nullable 警告との付き合い方

- .editorconfig
- Roslyn analyzer
- StyleCop / FxCop 系 analyzer
- nullable warning
- warning を抑制する判断
- チームで揃えるルール

### Part 10: パフォーマンスと落とし穴

#### allocation を意識する場面

- allocation の基本
- GC
- hot path
- struct の使いすぎ
- pooling
- Span / Memory

#### boxing / unboxing

- boxing
- unboxing
- interface と boxing
- generic による回避
- hidden allocation

#### LINQ の過剰使用

- 読みやすさとコスト
- 多重列挙
- 中間コレクション
- for ループに戻す判断
- DB LINQ と LINQ to Objects の違い

#### 文字列処理のコスト

- string allocation
- interpolation
- concatenation
- StringBuilder
- formatting
- culture

#### async の落とし穴

- sync-over-async
- deadlock
- fire-and-forget
- unobserved exception
- 過剰な async
- cancellation 漏れ

#### 例外を制御フローに使わない

- 例外のコスト
- TryParse pattern
- Result 型的な戻り値
- validation error
- expected error と unexpected error

#### BenchmarkDotNet の入口

- micro benchmark
- BenchmarkDotNet
- 計測前に疑うこと
- Release build
- JIT と warmup
- ベンチマーク結果の読み方

### Appendix: その他の実務定石

#### プロセス起動

- ProcessStartInfo
- 標準出力と標準エラー
- exit code
- shell execute
- コマンドインジェクションへの注意

#### 環境変数

- Environment.GetEnvironmentVariable
- 実行環境ごとの設定
- secret の扱い
- テストでの差し替え

#### バージョン情報

- assembly version
- file version
- informational version
- Git commit hash
- アプリケーション表示用のバージョン

#### ZIP 操作

- ZipFile
- ZipArchive
- entry の読み書き
- 一時ディレクトリ
- zip slip への注意

#### reflection

- Type
- PropertyInfo
- Attribute
- reflection のコスト
- trimming / AOT との相性

#### source generator の入口

- source generator の用途
- incremental generator
- 生成コードの確認
- analyzer との関係
- 導入しすぎない判断

#### unsafe / native interop の入口

- unsafe code
- pointer
- P/Invoke
- NativeAOT
- 安全性と保守性の注意

## 執筆方針

- C# 8 以降の現代的な書き方を中心にする
- 古い書き方も、現場で遭遇しやすいものは比較対象として扱う
- サンプルコードは短く、意図が分かる単位にする
- 「よい例」だけでなく「避けたい例」も載せる
- 仕様説明で終わらせず、実務での判断基準を書く
- テストしやすさ、保守性、レビュー観点をできるだけ併記する
- パフォーマンスの話は、必要な場面と過剰最適化の境界も書く

## 今後の進め方

1. 各項目のサンプルコードを、より実務に近い例へ育てる
2. コードレビューで使えるチェックリストを各 Part に追加する
3. 重要トピックから、良い例 / 避けたい例 / 改善例の 3 点セットにする
4. 実務で遭遇した落とし穴を随時追記する
5. 必要に応じて項目単位の独立ファイルへ分割する

