# Part 15 チェックリスト: 運用・品質・配布補足

このチェックリストは、PR、リリース前確認、運用設計レビューで、観測性、health checks、CI 品質ゲート、NuGet 配布、secret 運用を確認するために使います。すべてを毎回見る必要はありませんが、本番影響がある変更、共通基盤、外部公開 API、認証情報を扱う変更では確認を深くします。

## 観測性

- [ ] 重要な処理に、ログ、metrics、traces のどれで確認するかが決まっているか。
- [ ] ログだけに依存せず、成功率、失敗率、処理時間、キュー長などの傾向を metric で見られるか。
- [ ] trace で、入口処理、DB、HTTP、メッセージ処理などの境界が追えるか。
- [ ] span 名や metric 名が、実装詳細ではなく運用者が理解できる単位になっているか。
- [ ] user id、request id、raw path、email など高 cardinality または機密性のある値を metric label に入れていないか。
- [ ] error、retry、timeout、cancellation を、ログと trace の両方で区別できるか。

## Health checks

- [ ] liveness はプロセスが再起動すべき状態かを確認するだけに絞られているか。
- [ ] readiness はリクエストを受けてよい状態かを確認し、依存先や初期化状態を必要に応じて見るか。
- [ ] startup check が必要な重い初期化を、liveness や readiness と混ぜていないか。
- [ ] DB や外部 API の一時障害で、コンテナ再起動ループを起こさない設計になっているか。
- [ ] health endpoint が詳細な内部情報、connection string、secret を返していないか。
- [ ] 監視、ロードバランサー、Kubernetes probe など、利用者ごとの期待値に合っているか。

## CI 品質ゲート

- [ ] formatter、analyzer、test、coverage、package validation のどれを必須にするか明確か。
- [ ] analyzer の error 化は、既存負債と導入順序を考慮しているか。
- [ ] ローカルで CI と同じコマンドを再現できるか。
- [ ] test は単体テスト、結合テスト、遅いテストの実行タイミングが分かれているか。
- [ ] coverage は数字だけでなく、重要な分岐や失敗時の挙動を守れているか。
- [ ] flaky test、外部サービス依存、時刻依存、並列実行依存の扱いが決まっているか。

## NuGet と配布

- [ ] version 更新が SemVer に沿い、breaking change が minor や patch に混ざっていないか。
- [ ] public API の削除、シグネチャ変更、nullable 注釈変更、例外変更が利用者影響として確認されているか。
- [ ] package metadata に README、license、repository、tags、icon、release notes が含まれているか。
- [ ] Source Link、symbols、XML documentation が利用者のデバッグや補完に役立つ状態か。
- [ ] 依存 package の version 範囲が広すぎず狭すぎず、利用者側の解決を壊しにくいか。
- [ ] prerelease、internal package、public package の配布先と権限が分かれているか。

## 設定と secret

- [ ] secret を `appsettings.json`、ソースコード、テストデータ、README、CI ログに置いていないか。
- [ ] 開発、本番、CI で secret の注入経路が分かれているか。
- [ ] Options validation で、必須設定、範囲、形式、矛盾を起動時に検出できるか。
- [ ] secret のローテーション時に、再起動、再読み込み、接続再作成の方針が決まっているか。
- [ ] 設定値や secret を例外メッセージ、ログ、health check response に出していないか。
- [ ] 権限は最小化され、アプリが必要な secret だけを読めるか。

## リリース前確認

- [ ] 新しい metric、trace、health check、CI rule、package 設定が運用ドキュメントに反映されているか。
- [ ] 監視アラートは、検知したい利用者影響に対応しているか。
- [ ] 失敗時の rollback、feature flag、configuration rollback の手段があるか。
- [ ] パッケージや設定の変更に、互換性と移行手順が書かれているか。
- [ ] 本番で異常が起きたとき、どの dashboard、trace、ログ、runbook を見るか説明できるか。

---

[Part README に戻る](README.md)
