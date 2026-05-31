# TODO: ドキュメント拡充候補

最終更新: 2026-05-31

複数サブエージェントで全 Part と Appendix を監査した結果です。優先度は次の基準です。

- P0: セキュリティ、運用事故、データ不整合、公開 API 破壊など、実務影響が大きいもの。
- P1: 実務判断、レビュー、テストの具体性が大きく上がるもの。
- P2: 導線、網羅性、補助的な深掘りとして有用なもの。

## 次に着手するなら

1. Part 10〜11 の Web API / EF Core セキュリティと本番運用
2. Part 5 / Appendix のファイル、ZIP、プロセス起動まわりの安全性
3. Part 7〜8 の timeout / retry / cancellation / background worker の失敗設計
4. Part 15 の OpenTelemetry、secret、CI、NuGet 配布の運用具体化
5. Part 6 / 9 の層境界、transaction、outbox、ProblemDetails の横断設計

## Part 1: 現代 C# の基礎

- [ ] P0: `docs/part-01-modern-csharp-basics/csharp-と-dotnet-の全体像.md`
  - SDK / Runtime / TFM に加え、LTS/STS、`global.json`、CI SDK 固定、multi-targeting、nullable/analyzer を warnings as errors にする判断を追加する。
  - 実務の導入判断に必要な運用・保守観点を補う。
- [ ] P0: `docs/part-01-modern-csharp-basics/property-init-required-readonly.md`
  - C# 14 `field` backed properties、`required` と constructor / serializer / configuration binding の相性、`SetsRequiredMembers` の注意点を追加する。
  - `required` を validation と誤解しやすい点を具体例で補強する。
- [ ] P0: `docs/part-01-modern-csharp-basics/class-struct-record-record-struct.md`
  - record の浅い不変性、配列や mutable collection を持つ record の等価性、`with` が validation を迂回し得る例を追加する。
- [ ] P1: `docs/part-01-modern-csharp-basics/exception-using-idisposable-リソース管理.md`
  - cancellation と例外を混ぜない判断、exception filter、`OperationCanceledException`、async dispose 失敗時の扱いを追加する。
- [ ] P2: `docs/part-01-modern-csharp-basics/namespace-using-file-scoped-namespace.md`
  - SDK の implicit global usings、C# 12 alias any type、file-local type の配置方針を追加する。

## Part 2: 基本文法の実務イディオム

- [ ] P0: `docs/part-02-basic-idioms/enum-flags-enum-代替設計.md`
  - `Enum.IsDefined`、未定義値、JSON/DB 永続化、`JsonStringEnumConverter`、EF Core value conversion、リネーム移行の例を追加する。
- [ ] P1: `docs/part-02-basic-idioms/null-処理.md` と `docs/part-01-modern-csharp-basics/型-値型-参照型-nullable-reference-types.md`
  - 重複を整理し、Part 1 は nullable contract / 型設計、Part 2 は `??`、`?.`、境界での正規化、Result/Option 代替に寄せる。
- [ ] P1: `docs/part-02-basic-idioms/range-index-tuple-deconstruction-匿名型.md`
  - collection expression の target type、spread `..` のコピーコスト、C# 12 alias any type、tuple alias の使いどころを追加する。
- [ ] P1: `docs/part-02-basic-idioms/if-switch-pattern-matching.md`
  - sealed record hierarchy による疑似 discriminated union、switch の網羅性、`when` guard の使いすぎ、状態追加時のテスト観点を追加する。
- [ ] P2: `docs/part-01-modern-csharp-basics/checklist.md`、`docs/part-02-basic-idioms/checklist.md`、`docs/part-03-advanced-csharp-language/checklist.md`
  - Part 1〜3 の本文拡充後にチェックリストへ反映する。

## Part 3: C# 言語機能のステップアップ

- [ ] P0: `docs/part-03-advanced-csharp-language/primary-constructors-and-modern-syntax.md`
  - C# 13/14 の具体例を追加する。
  - `params` collections、`System.Threading.Lock`、C# 14 extension members、null-conditional assignment、`field` backed properties を「採用してよい場面 / 保留すべき場面」で整理する。
- [ ] P0: `docs/part-03-advanced-csharp-language/extension-methods.md`
  - C# 14 extension members の章を追加し、従来の extension method との使い分け、extension property を公開 API にするリスク、namespace 汚染のレビュー観点を補う。
- [ ] P0: `docs/part-03-advanced-csharp-language/generics-variance-constraints.md`
  - generic math、`static abstract` interface member、`INumber<T>` 系、`ref struct` と generics の制約を追加する。
- [ ] P1: `docs/part-03-advanced-csharp-language/attributes.md`
  - `[Experimental]`、trimming/AOT 関連属性、source generator 入力としての attribute、起動時検証のサンプルを増やす。
- [ ] P1: `docs/part-03-advanced-csharp-language/iterators-yield.md`
  - `IAsyncEnumerable<T>` と `Task<List<T>>` / `Channel<T>` の使い分け、EF Core `IQueryable` との違い、キャンセル伝播の例を追加する。
- [ ] P1: `docs/part-03-advanced-csharp-language/delegates-events.md`
  - 非同期 callback の `Func<..., ValueTask>`、`CancellationToken` を含む delegate 設計、event より mediator/channel を選ぶ場面を追加する。

## Part 4: コレクションと LINQ

- [ ] P1: `docs/part-04-collections-linq/array-list-dictionary-hashset.md`
  - `ConcurrentDictionary`、immutable collection、`FrozenDictionary` / `FrozenSet`、`Queue` / `Stack` の使いどころを補う。
- [ ] P1: `docs/part-04-collections-linq/linq-で避けたい書き方とパフォーマンス.md`
  - `IQueryable` と `IEnumerable` の境界、EF Core の N+1、projection、`Include`、client evaluation、発行 SQL 確認を拡充する。
- [ ] P2: `docs/part-04-collections-linq/linq-の基本-遅延評価-即時評価.md`
  - `IAsyncEnumerable<T>`、async streaming、キャンセル、複数回列挙できない stream の注意を追加する。
- [ ] P2: `docs/part-04-collections-linq/linq-を使いこなす.md`
  - `GroupBy`、`DistinctBy`、`ToDictionary` の comparer 指定と代表要素の決まり方を例で補強する。
- [ ] P2: `docs/part-04-collections-linq/checklist.md`
  - チェックリストで先行している論点を本文へのリンクや「詳しくは各章へ」で接続する。

## Part 5: 文字列・日付・ファイル・データ形式

- [ ] P0: `docs/part-05-strings-dates-files-data/file-path-directory-操作.md`
  - パストラバーサル対策を、OS ごとの大文字小文字、区切り文字、シンボリックリンク・junction・hard link まで含めて拡充する。
- [ ] P0: `docs/part-05-strings-dates-files-data/file-path-directory-操作.md`
  - 一時ファイル、原子的な置き換え、`FileMode.CreateNew`、部分書き込み、上書き競合、後始末の例を追加する。
- [ ] P0: `docs/part-05-strings-dates-files-data/タイムゾーンと-utc.md`
  - Windows ID と IANA ID の扱い、Linux 配置、変換ライブラリや設定方針を具体例つきで補う。
- [ ] P1: `docs/part-05-strings-dates-files-data/datetime-dateonly-timeonly-datetimeoffset.md`
  - 期間条件を半開区間 `start <= x && x < end` で扱う例、DB 精度・丸め・月末日末を避ける例を追加する。
- [ ] P1: `docs/part-05-strings-dates-files-data/datetime-dateonly-timeonly-datetimeoffset.md`
  - `Parse` / `TryParseExact`、`CultureInfo.InvariantCulture`、入力形式の固定、失敗時の扱いを拡充する。
- [ ] P1: `docs/part-05-strings-dates-files-data/json-serialization-と-system-text-json.md`
  - 大きな JSON、streaming、`Utf8JsonReader`、source generation、AOT 対応の判断を追加する。
- [ ] P1: `docs/part-05-strings-dates-files-data/json-serialization-と-system-text-json.md`
  - unknown property、`JsonExtensionData`、欠落値、`required`、nullable、既定値の互換性例を増やす。
- [ ] P1: `docs/part-05-strings-dates-files-data/string-比較-検索-分割-補間.md`
  - Unicode 正規化、全角半角、結合文字、`Length` と見た目の文字数の違い、検索・ID 正規化の境界を追加する。
- [ ] P1: `docs/part-05-strings-dates-files-data/正規表現と-generatedregex.md`
  - `RegexOptions.NonBacktracking`、`\A` / `\z`、timeout の共通方針、ReDoS テスト例を追加する。
- [ ] P2: `docs/part-05-strings-dates-files-data/stringbuilder-span-memory-の入口.md`
  - `Span<T>` を使う前に `Memory<T>`、`ArrayPool<T>`、`string.Create`、通常 API のどれを選ぶかの境界を補う。
- [ ] P2: `docs/part-05-strings-dates-files-data/checklist.md`
  - チェックリストで先行している論点を本文へのリンクや「詳しくは各章へ」で接続する。

## Part 6: オブジェクト指向と設計パターン

- [ ] P0: `docs/part-06-oop-patterns/value-object-entity-dto.md`
  - DTO、Entity、Value Object がドメイン層・アプリケーション層・インフラ層のどこに属するか、入力 DTO から Value Object を生成して Entity に渡す流れ、出力 DTO への mapping 例を追加する。
- [ ] P0: `docs/part-06-oop-patterns/dependency-injection.md`
  - Composition Root、アプリケーション層の port/interface、インフラ実装の登録、ドメイン層へ DI コンテナを持ち込まない例を追加する。
- [ ] P0: `docs/part-06-oop-patterns/README.md`
  - 「ドメインモデルとアプリケーション層の境界」を重要トピックに追加し、Value Object / Entity / DTO、DI、SOLID へどう接続するかを読む順番に明記する。
- [ ] P0: 新規候補 `docs/part-06-oop-patterns/domain-model-application-boundary.md`
  - ドメインモデル、アプリケーションサービス、ドメインサービス、Repository interface、DTO、EF Core entity の責務分担を 1 章で整理する。
- [ ] P1: `docs/part-06-oop-patterns/solid-原則を-csharp-コードで読む.md`
  - SRP/OCP/LSP/ISP/DIP それぞれに短い「避けたい例」と「改善例」を追加する。
- [ ] P1: `docs/part-06-oop-patterns/interface-と-abstract-class-の使い分け.md`
  - interface を port として使う場合、公開 API と内部 interface の違い、interface 追加・変更時の互換性、default interface method の注意を拡充する。
- [ ] P1: `docs/part-06-oop-patterns/template-method-strategy-factory.md`
  - Strategy / Factory / Template Method の選択表、単純な `switch` で十分な場合、DI 登録と選択ロジックの置き場所を追加する。
- [ ] P1: `docs/part-06-oop-patterns/カプセル化-継承-ポリモーフィズム.md`
  - anemic domain model、状態遷移メソッド、コレクション内の不変条件、継承より委譲を選ぶ改善例を追加する。
- [ ] P1: `docs/part-06-oop-patterns/checklist.md`
  - ドメイン層が DTO / EF Core / DI コンテナ / HTTP に依存していないか、アプリケーション層が transaction と外部 I/O の境界を持てているかを追加する。
- [ ] P2: `docs/part-06-oop-patterns/template-method-strategy-factory.md`
  - Decorator、Adapter、Specification、Repository など、実務で頻出する関連パターンを「詳説する / しない」の整理として追加する。
- [ ] P2: `docs/part-06-oop-patterns/dependency-injection.md`
  - `IOptions<T>` / `IOptionsSnapshot<T>` / `IOptionsMonitor<T>`、`HttpClientFactory`、起動時 DI 検証、ライフタイム不整合のテスト例を補足する。

## Part 7: 非同期・並列・外部 I/O

- [ ] P0: `docs/part-07-async-parallel-io/タイムアウト-リトライ-キャンセル設計.md`
  - per-attempt timeout、全体 deadline、呼び出し元キャンセルの切り分け例を追加する。
- [ ] P0: `docs/part-07-async-parallel-io/httpclient-の定石.md`
  - 429/503、`Retry-After`、指数バックオフ、jitter、冪等性キーを含む retry policy の例を追加する。
- [ ] P0: `docs/part-07-async-parallel-io/並列処理-parallel-plinq-channel.md`
  - PLINQ の専用節を追加し、`AsParallel`、`WithDegreeOfParallelism`、`AsOrdered`、副作用禁止、例外集約を扱う。
- [ ] P1: `docs/part-07-async-parallel-io/async-await-の基本.md`
  - `Task.WhenAll` の一部失敗時に入力 ID と例外を対応付ける例、兄弟タスクのキャンセル方針を追加する。
- [ ] P1: `docs/part-07-async-parallel-io/task-valuetask-cancellationtoken.md`
  - `ValueTask` の複数 await 禁止、保存して後で await しない、`AsTask()` の判断例を追加する。
- [ ] P1: `docs/part-07-async-parallel-io/httpclient-の定石.md`
  - 大きなレスポンス向けに `HttpCompletionOption.ResponseHeadersRead`、stream dispose、全量読み込み回避を追加する。
- [ ] P1: `docs/part-07-async-parallel-io/checklist.md`
  - 高度な項目から、本文未収録の項目へリンクまたは本文化する。
- [ ] P2: `docs/part-07-async-parallel-io/configureawait-の現代的な扱い.md`
  - `ConfigureAwait(false)` と `AsyncLocal`、Culture、logging scope が完全に切れるわけではない点を小例で補足する。

## Part 8: 並行処理・バックグラウンド処理補足

- [ ] P0: `docs/part-08-concurrency-background-services/periodicTimer-backgroundService.md`
  - `BackgroundService` で scoped service を使う場合の `IServiceScopeFactory` 例を追加する。
- [ ] P0: `docs/part-08-concurrency-background-services/producer-consumer-channel.md`
  - consumer 失敗時の retry、dead-letter、`TryComplete(exception)`、shutdown drain の方針を追加する。
- [ ] P1: `docs/part-08-concurrency-background-services/semaphoreSlim-throttling.md`
  - `SemaphoreSlim(initialCount, maxCount)`、over-release 検出、巨大入力で全 task を先に作る問題と `Channel` / batching への切替基準を追加する。
- [ ] P1: `docs/part-08-concurrency-background-services/lock-monitor-thread-safety.md`
  - `Interlocked`、`Volatile`、`ReaderWriterLockSlim` を使う判断と、`lock` で十分な場面の比較を追加する。
- [ ] P1: `docs/part-08-concurrency-background-services/concurrent-collections.md`
  - `AddOrUpdate` の factory 複数回実行、列挙のスナップショット性、read-modify-write の非原子性を例で追加する。
- [ ] P1: `docs/part-08-concurrency-background-services/producer-consumer-channel.md`
  - `SingleReader`、`SingleWriter`、複数 consumer、順序保証の崩れ方を追加する。
- [ ] P1: `docs/part-08-concurrency-background-services/checklist.md`
  - 高度な項目から、本文未収録の項目へリンクまたは本文化する。
- [ ] P2: `docs/part-08-concurrency-background-services/periodicTimer-backgroundService.md`
  - fake clock、短い interval、停止 token、例外時継続の検証例を追加する。

## Part 9: 実務アプリケーションの定石

- [ ] P0: `docs/part-09-application-practices/ログと構造化ログ.md`
  - OpenTelemetry、`Activity`、trace id、metrics、health check、ログ・メトリクス・トレースの使い分けを追加する。
- [ ] P0: `docs/part-09-application-practices/エラー処理と例外設計.md`
  - `ProblemDetails` への一元変換、HTTP status と error code の対応表、trace id の返し方を追加する。
- [ ] P0: `docs/part-09-application-practices/repository-service-usecase-分離.md`
  - UseCase が持つ transaction 境界、外部 API 呼び出し順、outbox / Unit of Work の判断例を拡充する。
- [ ] P1: `docs/part-09-application-practices/設定と-options-pattern.md`
  - named options、`IValidateOptions<T>`、`PostConfigure`、reload 時の `IOptionsMonitor` 注意点を追加する。
- [ ] P1: `docs/part-09-application-practices/validation.md`
  - model binding 失敗、invalid JSON、nullable、DB 一意制約との競合、async validation の扱いを追加する。
- [ ] P1: `docs/part-09-application-practices/web-api-での-dto-境界-マッピング.md`
  - pagination、filtering、sorting、ETag / concurrency、idempotency key、versioning の具体例を追加する。
- [ ] P1: `docs/part-09-application-practices/db-アクセス時の注意点.md`
  - `DbContext` lifetime、thread safety、connection pooling、retry / execution strategy、command timeout、split query を追加する。
- [ ] P2: `docs/part-09-application-practices/checklist.md`
  - テスト観点を独立セクション化し、unit / integration / contract / migration / observability test を追加する。
- [ ] P2: `docs/part-09-application-practices/README.md`
  - 「実務アプリの境界設計」と「観測性」を重要トピックに明示する。
- [ ] P2: `docs/part-09-application-practices/db-アクセス時の注意点.md`
  - raw SQL、bulk update/delete、tenant 境界、監査項目更新漏れの例を追加する。

## Part 10: ASP.NET Core / Web API 実務補足

- [ ] P0: `docs/part-10-aspnetcore-webapi/authentication-authorization.md`
  - JWT bearer の検証設定、issuer/audience、clock skew、scope/role claim mapping、fallback policy、`AllowAnonymous` の扱いを追加する。
- [ ] P0: `docs/part-10-aspnetcore-webapi/authentication-authorization.md`
  - cookie 認証時の CSRF、SameSite、SPA/BFF 構成、API key の保管・ローテーションを追加する。
- [ ] P0: `docs/part-10-aspnetcore-webapi/problemdetails-error-responses.md`
  - validation error、model binding error、認証/認可失敗、例外ハンドラの統合テスト例を追加する。
- [ ] P0: `docs/part-10-aspnetcore-webapi/rate-limit-cors-health-checks.md`
  - CORS の credentials、preflight、`AllowAnyOrigin` との危険な組み合わせ、環境別 origin 管理を詳述する。
- [ ] P1: `docs/part-10-aspnetcore-webapi/middleware-filters-endpoint-filters.md`
  - `UseCors`、`UseAuthentication`、`UseAuthorization`、`UseRateLimiter`、`UseExceptionHandler` の順序例を拡充する。
- [ ] P1: `docs/part-10-aspnetcore-webapi/middleware-filters-endpoint-filters.md`
  - MVC filter の種類、endpoint filter の validation 例、短絡時の ProblemDetails 例を追加する。
- [ ] P1: `docs/part-10-aspnetcore-webapi/minimal-api-controller-boundaries.md`
  - typed results、OpenAPI metadata、API versioning、route group 単位の契約整理を追加する。
- [ ] P1: `docs/part-10-aspnetcore-webapi/checklist.md`
  - `WebApplicationFactory` による認証・認可・ProblemDetails・CORS・rate limit の統合テスト項目を追加する。
- [ ] P2: `docs/part-10-aspnetcore-webapi/rate-limit-cors-health-checks.md`
  - `Retry-After`、rate limiter の queue 設定、IP spoofing / proxy 配下の client id 取り扱いを追加する。
- [ ] P2: `docs/part-10-aspnetcore-webapi/rate-limit-cors-health-checks.md`
  - health endpoint の公開範囲、認証要否、レスポンス内容の秘匿、liveness/readiness/startup の使い分けを追加する。

## Part 11: EF Core / DB アクセス深掘り

- [ ] P0: `docs/part-11-efcore-data-access/transactions-concurrency.md`
  - `DbUpdateConcurrencyException` の具体的な処理、409 への変換、再読み込み、利用者への競合表示例を追加する。
- [ ] P0: `docs/part-11-efcore-data-access/migrations-and-schema-changes.md`
  - expand-contract、後方互換 migration、idempotent script / migration bundle、本番適用手順、バックアップ・復旧を追加する。
- [ ] P0: `docs/part-11-efcore-data-access/raw-sql-and-injection.md`
  - `ExecuteSqlInterpolated`、`ExecuteUpdate/Delete` との使い分け、権限最小化、監査ログ、機密値ログ出力の注意を追加する。
- [ ] P1: `docs/part-11-efcore-data-access/tracking-no-tracking-projection.md`
  - `AsNoTrackingWithIdentityResolution`、`Attach` / `Update` の過剰更新、DTO からの部分更新、overposting 対策を追加する。
- [ ] P1: `docs/part-11-efcore-data-access/include-query-shape-n-plus-one.md`
  - filtered include、split query の整合性・往復回数の trade-off、ページングと Include の組み合わせを追加する。
- [ ] P1: `docs/part-11-efcore-data-access/checklist.md`
  - Testcontainers / 実 DB provider、SQL snapshot、クエリ回数検証、制約違反・同時更新テストを追加する。
- [ ] P2: `docs/part-11-efcore-data-access/repository-pattern-with-efcore.md`
  - specification/query object の境界、`DbContext` を直接使う場合のテスト方針を追加する。
- [ ] P2: `docs/part-11-efcore-data-access/README.md`
  - multi-tenancy、soft delete、global query filter、tenant 漏れの注意を新規章候補として追加する。

## Part 12: テストしやすい C#

- [ ] P0: `docs/part-12-testable-csharp/単体テストしやすい設計.md`
  - unit test・integration test・contract test の使い分け、テストピラミッド、DB/API 境界で何をどこまで保証するかを追記する。
- [ ] P0: `docs/part-12-testable-csharp/時刻-乱数-外部-i-o-の分離.md`
  - `HttpClient`、`HttpMessageHandler`、ファイル I/O、環境変数、Options、外部 API の timeout/retry/429/5xx をテストする具体例を追加する。
- [ ] P0: `docs/part-12-testable-csharp/例外・非同期・境界値のテスト.md`
  - `Task.WhenAll`、fire-and-forget、background service、`IAsyncEnumerable<T>`、`ValueTask`、並行処理の失敗・キャンセル・部分成功のテスト観点を追加する。
- [ ] P1: `docs/part-12-testable-csharp/xunit-nunit-mstest-の観点.md`
  - xUnit/NUnit/MSTest の属性、fixture lifecycle、並列実行、collection fixture、parameterized test、assertion 差分を比較表で追加する。
- [ ] P1: `docs/part-12-testable-csharp/mock-stub-fake-の使い分け.md`
  - strict/loose mock、Moq/NSubstitute などの読みやすさ、`ILogger` 検証、外部 SDK を直接 mock しない例、mock 過多の判断基準を追加する。
- [ ] P1: `docs/part-12-testable-csharp/テストデータビルダー.md`
  - nested aggregate、collection、immutable record、builder 再利用による状態漏れ、AutoFixture 等との使い分けを追加する。
- [ ] P1: `docs/part-12-testable-csharp/checklist.md`
  - flaky test の原因別チェック、並列実行、共有 DB、時計、ランダム、外部 API、順序依存の切り分け項目を独立節として追加する。

## Part 13: スタイル、レビュー、保守性

- [ ] P0: `docs/part-13-style-review-maintainability/analyzer-editorconfig-nullable-警告との付き合い方.md`
  - `.editorconfig` の severity 方針、`TreatWarningsAsErrors`、CI で落とす警告、IDE suggestion に留める警告の分類例を追加する。
- [ ] P0: `docs/part-13-style-review-maintainability/analyzer-editorconfig-nullable-警告との付き合い方.md`
  - nullable attributes の具体例として `NotNullWhen`、`MemberNotNull`、`MaybeNull`、`DisallowNull`、`required`、JSON/DB 境界での null 注釈を追加する。
- [ ] P1: `docs/part-13-style-review-maintainability/コメントと-xml-documentation.md`
  - `<param>`、`<returns>`、`<exception>`、`<remarks>`、`<inheritdoc>`、`<example>` の使い分け、CS1591 の扱いを追加する。
- [ ] P1: `docs/part-13-style-review-maintainability/コードレビュー観点.md`
  - PR description の確認項目、blocker にする条件、DB migration/API compatibility/serialization 変更のレビュー観点を追加する。
- [ ] P1: `docs/part-13-style-review-maintainability/命名規則.md`
  - 単位付き命名、nullable を示す名前、collection の単数複数、`Try` / `Find` / `Get` の失敗時契約、`CancellationToken` 付き async API の命名例を追加する。
- [ ] P2: `docs/part-13-style-review-maintainability/読みやすい条件式.md`
  - pattern matching、switch expression、enum 追加時の未対応検知、複雑な権限条件のテスト例を追加する。
- [ ] P2: `docs/part-13-style-review-maintainability/メソッド分割-責務分割.md`
  - private method 分割で済ませる場合と型抽出する場合、transaction/logging/notification の境界を分ける判断例を追加する。

## Part 14: パフォーマンスと落とし穴

- [ ] P1: `docs/part-14-performance-pitfalls/benchmarkdotnet-の入口.md`
  - `MemoryDiagnoser`、`Params`、`GlobalSetup`、`Consumer`、baseline、結果表の読み方、環境情報の貼り方を追加する。
- [ ] P1: `docs/part-14-performance-pitfalls/allocation-を意識する場面.md`
  - `ArrayPool<T>`、`Span<T>`、`Memory<T>`、object pooling の所有権、返却、clear、例外時返却の短い例を追加する。
- [ ] P1: `docs/part-14-performance-pitfalls/async-の落とし穴.md`
  - `Task.WhenAll` の同時実行数制御、timeout、部分失敗、`IAsyncEnumerable` / Channel の backpressure を追加する。
- [ ] P1: `docs/part-14-performance-pitfalls/文字列処理のコスト.md`
  - `Regex`、`Split`、`Substring`、`StringBuilder` の使い分け、ログ無効時の JSON serialize 回避を拡充する。
- [ ] P1: `docs/part-14-performance-pitfalls/linq-の過剰使用.md`
  - `List<T>.Contains` と `HashSet<T>`、`OrderBy` / `GroupBy` / `Distinct`、`AsEnumerable()` 後の DB 外処理を追加する。
- [ ] P2: `docs/part-14-performance-pitfalls/boxing-unboxing.md`
  - enum formatting、interface constrained generic、`IEquatable<T>`、custom comparer の boxing 回避例を追加する。
- [ ] P2: `docs/part-14-performance-pitfalls/例外を制御フローに使わない.md`
  - retry すべき例外、即失敗すべき例外、例外フィルター、`throw;` と `throw ex;` の差を追加する。
- [ ] P2: `docs/part-14-performance-pitfalls/checklist.md`
  - GC heap dump、`dotnet-counters`、`dotnet-trace`、profiler など実アプリ計測の入口を追加する。

## Part 15: 運用・品質・配布補足

- [ ] P0: `docs/part-15-operations-quality-delivery/metrics-tracing-opentelemetry.md`
  - ASP.NET Core での OpenTelemetry 最小構成、`ResourceBuilder`、exporter、collector、trace-log correlation、semantic conventions の例を追加する。
- [ ] P0: `docs/part-15-operations-quality-delivery/metrics-tracing-opentelemetry.md`
  - SLI/SLO、alert、dashboard、runbook までの設計観点を追加する。
- [ ] P0: `docs/part-15-operations-quality-delivery/configuration-and-secret-operations.md`
  - secret scanning、CI ログマスク、Key Vault 等の権限最小化、ローテーション時の再読み込み方針を具体化する。
- [ ] P0: `docs/part-15-operations-quality-delivery/ci-analyzers-formatters-tests.md`
  - `dotnet format`、analyzer、nullable、test category、coverage、package validation をローカル再現可能な品質ゲートとして整理する。
- [ ] P0: `docs/part-15-operations-quality-delivery/nuget-versioning-library-design.md`
  - `dotnet pack`、`dotnet nuget verify`、API compatibility、README/license/icon/release notes、symbols/Source Link 検証を追加する。
- [ ] P1: `docs/part-15-operations-quality-delivery/health-checks-readiness-liveness.md`
  - timeout、degraded、依存先別 readiness、Kubernetes probe 設定例、migration/warm-up 中の挙動を追加する。
- [ ] P1: `docs/part-15-operations-quality-delivery/checklist.md`
  - SLO/alert/runbook、secret scanning、API compatibility、package signing/SBOM を追記する。
- [ ] P2: `docs/part-15-operations-quality-delivery/README.md`
  - Part 14 との接続として「性能改善後に何を metric/trace/alert で見るか」を追加する。

## Appendix

- [ ] P0: `docs/appendix/プロセス起動.md`
  - キャンセル時の子プロセス終了、`Kill(entireProcessTree: true)`、標準入力、大容量 stdout/stderr の逐次読み取り、`ProcessStartInfo.Environment`、ログ時の引数マスクを追加する。
- [ ] P0: `docs/appendix/zip-操作.md`
  - ZIP bomb 対策として entry 数、総展開サイズ、読み取り時の byte count、同名・大小文字衝突、Windows 予約名、絶対パス、symlink 相当、temp 展開後の atomic move を追加する。
- [ ] P0: `docs/appendix/unsafe-native-interop-の入口.md`
  - `LibraryImport` と `DllImport` の使い分け、文字列 marshaling、`SetLastError`、`NativeLibrary.SetDllImportResolver`、pinning / `GCHandle`、endianness、`MemoryMarshal` の注意を拡充する。
- [ ] P1: `docs/appendix/環境変数.md`
  - ASP.NET Core configuration との関係、`__` による階層キー、優先順位、process/user/machine scope、コンテナ・CI・Kubernetes secret、reload 不可の前提を追加する。
- [ ] P1: `docs/appendix/バージョン情報.md`
  - `.csproj` の `Version` / `AssemblyVersion` / `FileVersion` / `InformationalVersion` 例、CI での commit hash 注入、SemVer と `System.Version` の差、container image tag / NuGet / API version の分離を追加する。
- [ ] P1: `docs/appendix/reflection.md`
  - trimming / NativeAOT 向けの `DynamicallyAccessedMembers`、`DynamicDependency`、`RequiresUnreferencedCode`、`NullabilityInfoContext`、`AssemblyLoadContext`、`TargetInvocationException` の扱いを追加する。
- [ ] P1: `docs/appendix/source-generator-の入口.md`
  - incremental generator の処理分解、`AdditionalTextsProvider`、`AnalyzerConfigOptions`、diagnostic location、`EmitCompilerGeneratedFiles`、generator テスト、NuGet packaging を追加する。
- [ ] P1: `docs/appendix/checklist.md`
  - プロセス起動・ZIP・native interop の P0 追加項目をチェックリストへ反映する。
- [ ] P2: 新規 `docs/appendix/一時ファイルと作業ディレクトリ.md`
  - temp directory、作業ディレクトリ、cleanup、atomic replace、権限、並列実行時の衝突を扱う章を追加する。
- [ ] P2: 新規 `docs/appendix/trimming-nativeaot-互換性.md`
  - reflection、source generator、JSON、DI、native interop を横断して trimming / NativeAOT 対応の判断基準を整理する。

## 全体索引・導線

- [ ] P1: `docs/index.md`
  - Part 一覧の下に「目的別索引」を追加し、外部コマンド、環境設定、配布物、メタプログラミング、native 境界、レビュー用チェックリストへ直接リンクする。
- [ ] P1: `README.md`
  - Appendix 行または読み方に、プロセス起動・環境変数・ZIP・reflection・source generator・unsafe/native interop を明記する。
- [ ] P2: `mkdocs.yml`
  - `全体索引` または `レビュー導線` の nav 項目を追加し、`docs/topic-index.md` や `docs/checklists.md` を置く案を検討する。

## 参考

- Microsoft Learn: C# language versioning  
  https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-versioning
- Microsoft Learn: What's new in C# 13  
  https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13
- Microsoft Learn: What's new in C# 14  
  https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-14
- Microsoft Learn: What's new in ASP.NET Core in .NET 10  
  https://learn.microsoft.com/en-us/aspnet/core/whats-new/
- Microsoft Learn: What's New in EF Core 10  
  https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/whatsnew
