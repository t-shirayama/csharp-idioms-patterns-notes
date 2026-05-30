# Part 6: 非同期・並列・外部 I/O

非同期処理は「速くするための魔法」ではなく、待ち時間の扱いを明確にして、スレッドやリソースを無駄に占有しないための設計です。並列処理は CPU を複数コアで使うための手段であり、外部 I/O は失敗や遅延を前提に境界を置く必要があります。

実務では、`async` / `await`、キャンセル、タイムアウト、リトライ、HTTP 呼び出し、並列化を別々の話としてではなく、「呼び出し元が待てるか」「中断できるか」「失敗時に再実行してよいか」という観点でつなげて考えます。

## 読み方

この Part では、非同期、並列、外部 I/O を「速くするテクニック」ではなく、待機、失敗、中断、同時実行数を設計する話として扱います。特に外部サービスや DB を含む処理では、正常系のコードだけでなく、遅延、キャンセル、リトライ、部分失敗をレビュー対象にします。

- I/O-bound ならまず `async` / `await` で待機中のスレッド占有を避ける。
- CPU-bound なら `Parallel`、PLINQ、`Task.Run` の必要性と上限を確認する。
- キャンセルは呼び出し元の意思、タイムアウトは待機時間の境界として分ける。
- リトライは冪等性、対象例外、全体の待機時間を決めてから入れる。
- 外部 I/O は専用クライアントに閉じ込め、HTTP や JSON の都合を業務コードへ漏らさない。

コードレビューでは、まず [Part 6 レビュー用チェックリスト](checklist.md) で async のつながり、`CancellationToken` の伝播、`HttpClient` のライフタイム、並列度の上限、retry の冪等性を確認します。正常系の読みやすさだけでなく、遅い、止めたい、失敗した、部分的に終わった、を説明できるかを基準にします。

## 項目一覧

- [async / await の基本](async-await-の基本.md)
- [Task、ValueTask、CancellationToken](task-valuetask-cancellationtoken.md)
- [ConfigureAwait の現代的な扱い](configureawait-の現代的な扱い.md)
- [HttpClient の定石](httpclient-の定石.md)
- [並列処理、Parallel、PLINQ、Channel](並列処理-parallel-plinq-channel.md)
- [タイムアウト、リトライ、キャンセル設計](タイムアウト-リトライ-キャンセル設計.md)
- [Part 6 レビュー用チェックリスト](checklist.md)

---

[章別ノート一覧に戻る](../index.md)
