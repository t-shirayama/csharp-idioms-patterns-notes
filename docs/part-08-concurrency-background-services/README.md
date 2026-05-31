# Part 8: 並行処理・バックグラウンド処理補足

Part 7 では `async` / `await`、外部 I/O、並列処理の入口を扱いました。この Part ではもう一歩進めて、複数の処理が同時に動くときの共有状態、排他、同時実行数、常駐処理、producer / consumer の設計を扱います。

並行処理は、単に「同時に動かす」だけでは安全になりません。実務では、誰が状態を持つのか、どこで順序を保証するのか、失敗時にどこまで止めるのか、終了時に残った作業をどう扱うのかを決める必要があります。

## このPartで身につくこと

- `lock`、`Monitor`、thread-safe collection を使う前に確認する共有状態の見方。
- `SemaphoreSlim` による同時実行数制御と、単純な排他との違い。
- `ConcurrentDictionary` などの concurrent collection を安全に使う判断基準。
- `PeriodicTimer` と `BackgroundService` による常駐処理の基本設計。
- `Channel` を使った producer / consumer と backpressure の考え方。
- 並行処理をレビューするときの、キャンセル、完了通知、例外、観測性の確認観点。

## 読む順番

最初に [lock / Monitor / thread safety](lock-monitor-thread-safety.md) を読み、共有 mutable state がどこにあるかを見つける視点を押さえます。次に [SemaphoreSlim による同時実行数制御](semaphoreSlim-throttling.md) を読み、排他と throttling の違いを整理します。

複数スレッドからコレクションを触る場面があるなら [Concurrent collections](concurrent-collections.md) を確認します。常駐処理や定期実行を扱う場合は [PeriodicTimer / BackgroundService](periodicTimer-backgroundService.md) を読み、最後に [producer / consumer と Channel](producer-consumer-channel.md) でキュー、完了通知、backpressure の設計に進みます。

## 重要トピック

- thread safety は型やメソッド単体ではなく、共有される状態と操作の単位で判断する。
- `lock` は短く、同期的なクリティカルセクションに限定する。
- `SemaphoreSlim` は外部 I/O や重い処理の同時実行数を制御するために使う。
- concurrent collection は単体操作を安全にするが、複数操作をまとめた不変条件までは守らない。
- `BackgroundService` は停止、例外、ログ、再起動、冪等性を含めて設計する。
- `Channel` は producer と consumer を分離し、bounded channel で投入側に backpressure をかけられる。

## 実務での使いどころ

Web API で大量の外部 API 呼び出しを同時に走らせる、ファイル取り込みをキュー化する、定期的に古いデータをクリーンアップする、複数ワーカーでジョブを処理する、といった場面でこの Part の知識が効きます。

特に、DB 接続数、外部 API の rate limit、メモリ使用量、アプリ終了時の処理継続を考える必要があるコードでは、単純な `Task.WhenAll` や `Parallel.ForEachAsync` だけでは不十分です。上限、待ち行列、停止、失敗時の扱いをコード上に表すことが重要です。

## よくある落とし穴

- `List<T>` や `Dictionary<TKey,TValue>` を複数スレッドから無防備に更新する。
- `lock` の中で `await` しようとする、または長い I/O を同期的に待つ。
- `ConcurrentDictionary` を使えば業務上の不変条件も自動で守られると思い込む。
- `Task.WhenAll(items.Select(...))` で入力件数分の I/O を一気に開始する。
- `BackgroundService` の例外、停止、キャンセル、アプリ終了時の残作業を決めていない。
- unbounded `Channel` に大量投入し、メモリを無制限に使う。

## レビュー観点

- 共有 mutable state はどこにあり、誰が更新するか明確か。
- 排他したいのは「1件ずつ処理」なのか「同時実行数をN件に制限」なのか。
- thread-safe collection の単体操作と、業務上の一連の操作を混同していないか。
- `CancellationToken` が待機、キュー投入、ワーカー処理、定期実行に伝播しているか。
- 例外時にワーカーが止まるのか、ログを出して継続するのか、再試行するのか決まっているか。
- backpressure、最大キュー長、処理遅延、破棄方針を説明できるか。

## 記事一覧

- [lock / Monitor / thread safety](lock-monitor-thread-safety.md): 共有状態、クリティカルセクション、排他の基本を整理します。
- [SemaphoreSlim による同時実行数制御](semaphoreSlim-throttling.md): 外部 I/O や重い処理の上限制御を扱います。
- [Concurrent collections](concurrent-collections.md): `ConcurrentDictionary` や `ConcurrentQueue` の使いどころと限界を確認します。
- [PeriodicTimer / BackgroundService](periodicTimer-backgroundService.md): 定期実行、常駐処理、停止処理の実務的な型を整理します。
- [producer / consumer と Channel](producer-consumer-channel.md): キュー、複数ワーカー、backpressure、完了通知を扱います。
- [Part 8 レビュー用チェックリスト](checklist.md): 並行処理とバックグラウンド処理をレビューできる一覧です。

---

[章別ノート一覧に戻る](../index.md)
