# Part 14 チェックリスト: パフォーマンスと落とし穴

このチェックリストは、パフォーマンス指摘を「速そうな書き方」ではなく、影響範囲、測定、保守性の判断に結びつけるために使います。hot path では深く確認し、通常の業務処理では読みやすさを優先できるかも同時に見ます。

## まず確認する前提

- [ ] そのコードは hot path、リクエストごとの共通処理、大量データ処理、UI の高頻度更新のどれに当たるか。
- [ ] 入力件数、データサイズ、呼び出し回数、同時実行数が将来増える可能性を確認しているか。
- [ ] 遅さの根拠が、ログ、メトリクス、プロファイラ、BenchmarkDotNet などで示されているか。
- [ ] 最適化により、正しさ、読みやすさ、テストしやすさがどれだけ悪化するかを見ているか。
- [ ] 変更前後で比較できる基準値や期待する改善幅があるか。
- [ ] 改善対象が、CPU、allocation、I/O 待ち、DB クエリ、ロック待ち、ネットワーク待ちのどれかに切り分けられているか。

## Allocation と GC

- [ ] ループ内や高頻度処理で、不要な `ToList()`、配列生成、一時 DTO、文字列生成が発生していないか。
- [ ] ラムダキャプチャ、closure、iterator、async state machine が、意図せず allocation を増やしていないか。
- [ ] `IEnumerable<T>` を返すために deferred execution を選んだ結果、呼び出し側で多重列挙や遅延例外を招いていないか。
- [ ] `struct` 化が、boxing、コピーコスト、default 値の扱いを悪化させていないか。
- [ ] interface 経由の呼び出し、非ジェネリック collection、`object` への格納で value type が boxing されていないか。
- [ ] `ArrayPool<T>`、object pooling、キャッシュの所有権、返却、例外時の扱いが明確か。
- [ ] pool から借りた buffer を返す前に、必要なら機密データや大きな参照を clear しているか。
- [ ] キャッシュに最大件数、期限、無効化条件、キーの粒度があり、メモリを抱え込み続けないか。
- [ ] allocation 削減のための `Span<T>`、`Memory<T>`、`ValueTask` が、寿命や再利用ルールを読みにくくしていないか。

## LINQ とコレクション操作

- [ ] `IEnumerable<T>`、`IReadOnlyList<T>`、`IQueryable<T>` のどれを扱っているか明確か。
- [ ] 同じ sequence に対して `Any()`、`Count()`、`Sum()`、`First()` などを重複実行していないか。
- [ ] `Where(...).ToList().Select(...).ToList()` のような不要な中間コレクションがないか。
- [ ] `IQueryable<T>` に、SQL へ変換できないメソッドや副作用を混ぜていないか。
- [ ] `OrderBy`、`GroupBy`、`Distinct`、`Contains` が、入力サイズ増加時に支配的なコストにならないか。
- [ ] `List<T>.Contains` を大量に呼ぶ箇所で、`HashSet<T>` へ切り替える根拠があるか。
- [ ] hot path で LINQ を使う場合、可読性の利益と allocation、delegate 呼び出し、多重列挙のコストを比べているか。
- [ ] DB 側で絞るべき処理を `AsEnumerable()` 後のメモリ上処理にしていないか。

## 文字列処理

- [ ] ループ内の `+` 連結が、件数増加時に大きな allocation にならないか。
- [ ] 多数の文字列を結合する場面で、`string.Join`、`StringBuilder`、補間文字列のどれが適切か検討しているか。
- [ ] ログ出力で、ログレベル無効時にも高コストな文字列整形、JSON serialize、例外生成をしていないか。
- [ ] logging では message template、source generator、`LoggerMessage` などを使う価値がある頻度か判断しているか。
- [ ] culture 依存の変換や比較で、`StringComparison` や `CultureInfo` を明示すべき箇所がないか。
- [ ] 大文字小文字変換後の比較ではなく、`StringComparison.OrdinalIgnoreCase` などで比較できないか。
- [ ] 正規表現、`Split`、`Substring` を高頻度に呼ぶ箇所で、compile、cache、`Span<T>` を使う理由と寿命を説明できるか。

## async と並行処理

- [ ] `async` 化が I/O 待ちを扱うためのもので、CPU bound 処理を隠していないか。
- [ ] `Task.Result`、`.Wait()`、同期 over 非同期により deadlock や thread pool 枯渇を招いていないか。
- [ ] `Task.Run` が ASP.NET などのサーバー側で、CPU 処理を別スレッドへ逃がすだけになっていないか。
- [ ] `Task.WhenAll` で並列化する処理に、同時実行数、外部サービス制限、timeout、失敗時の扱いがあるか。
- [ ] `CancellationToken` が、外部 I/O、長いループ、待機処理へ伝搬しているか。
- [ ] fire-and-forget が必要な場合、例外、再試行、ログ、アプリ終了時の扱いを説明できるか。
- [ ] `async void` がイベントハンドラー以外に使われていないか。
- [ ] `ValueTask` を使う場合、複数回 await、結果の保持、例外処理の制約を理解した上で選んでいるか。
- [ ] 非同期 stream や channel の利用で、backpressure、キャンセル、consumer 側の停止条件が明確か。

## 例外と失敗経路

- [ ] 通常の分岐や入力検証に、例外を制御フローとして使っていないか。
- [ ] `TryParse`、`TryGetValue`、検証結果型などで表した方が自然な失敗を例外にしていないか。
- [ ] hot path で例外オブジェクトを頻繁に生成し、stack trace 取得コストを見落としていないか。
- [ ] 例外を catch して握りつぶし、リトライ不能な状態、部分成功、不整合を隠していないか。
- [ ] リトライ対象の例外と即失敗すべき例外を区別しているか。
- [ ] 例外ログに機密情報や巨大な入力データを出していないか。
- [ ] 失敗時にも pool 返却、stream dispose、ロック解放などが保証されているか。
- [ ] `finally`、`await using`、`using` の境界が、非同期処理や早期 return でも期待どおりに働くか。

## 測定と BenchmarkDotNet

- [ ] micro benchmark の前に、実アプリでどこが遅いかを絞っているか。
- [ ] benchmark は Release build で実行され、JIT、warmup、tiered compilation、外れ値、allocation を考慮しているか。
- [ ] 入力サイズ、データ分布、失敗ケースが本番に近いか。
- [ ] PR に測定結果を載せる場合、.NET version、実行環境、コマンド、比較対象が分かるか。
- [ ] 平均時間だけでなく、P95/P99、allocation、GC 回数、I/O 待ち、DB クエリ数など、問題に合う指標を見ているか。
- [ ] benchmark 対象の処理が dead code elimination で消えないように、戻り値や `Consumer` を使っているか。
- [ ] 測定対象にログ、DB、ネットワーク、ファイル I/O を含めるか切り離すかを意図して決めているか。
- [ ] 差が小さい改善のために、理解しにくい実装へ置き換えていないか。

## 避けたいサイン

- [ ] 測定なしに「LINQ は遅い」「Span の方が速い」と断定している。
- [ ] `ToList()` が境界固定ではなく、なんとなく chain の途中に入っている。
- [ ] pooling や cache の導入で、所有権、寿命、無効化の方が難しくなっている。
- [ ] 例外、ログ、文字列整形が hot path のコストとして見落とされている。
- [ ] micro benchmark の改善だけを根拠に、実アプリのボトルネックから外れた箇所を複雑化している。

---

[Part README に戻る](README.md)
