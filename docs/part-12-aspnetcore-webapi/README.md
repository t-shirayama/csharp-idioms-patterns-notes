# Part 12: ASP.NET Core / Web API 実務補足

ASP.NET Core の Web API は、エンドポイントを書けるだけでは十分ではありません。認証、認可、エラー応答、横断処理、外部公開する契約、運用時のヘルスチェックまで含めて、アプリケーション境界として設計します。

この Part では、Web API 実務でレビュー対象になりやすい判断を扱います。重要なのは、フレームワーク機能をどれだけ知っているかではなく、「外から入る値をどこで受け止め、何を公開し、何を隠し、失敗をどう返すか」を一貫させることです。

## このPartで身につくこと

- authentication と authorization を分けて設計する考え方
- middleware、filter、endpoint filter の責務の置き場所
- `ProblemDetails` を使ったエラー応答の統一
- rate limit、CORS、health checks を本番運用の観点で読む力
- Minimal API と Controller の境界設計
- Web API のセキュリティ、テスト容易性、レビュー観点

## 読む順番

まず [Authentication / Authorization](authentication-authorization.md) で、誰かを確認する処理と、何を許可するかの処理を分けて理解します。

次に [Middleware / Filters / Endpoint Filters](middleware-filters-endpoint-filters.md) を読み、横断処理をどの層に置くかを整理します。

[ProblemDetails とエラー応答](problemdetails-error-responses.md) で失敗時の公開契約をそろえた後、[Rate Limit / CORS / Health Checks](rate-limit-cors-health-checks.md) で運用と防御の観点を補います。

最後に [Minimal API / Controller の境界](minimal-api-controller-boundaries.md) と [レビュー checklist](checklist.md) で、API の形をチームで判断できる状態にします。

## 重要トピック

- authentication は「利用者が誰か」、authorization は「何をしてよいか」を扱う
- policy based authorization で業務権限を名前付きの判断にする
- middleware はリクエスト全体、filter は MVC、endpoint filter は Minimal API の境界で使う
- エラー応答は status code、type、title、detail、trace id の責務を分ける
- CORS は認可ではなく、ブラウザーからの呼び出し制約として扱う
- health checks は liveness と readiness を分け、本番監視とデプロイ判断に使う
- Minimal API と Controller は好みではなく、複雑さ、契約、テスト、チーム運用で選ぶ

## 実務での使いどころ

新しい API を追加するときは、エンドポイントの実装だけでなく、認可ポリシー、入力検証、エラー応答、ログ、rate limit、テスト境界まで同時に確認します。API は外部との契約なので、あとから内部都合で変えるほど利用者への影響が大きくなります。

既存 API を改善するときは、まずレスポンスの一貫性と横断処理の置き場所を見ます。controller や handler に認可、例外変換、CORS、ログ、変換が散らばっていると、セキュリティレビューも障害調査も難しくなります。

## よくある落とし穴

- authentication が通れば何でもできる設計になっている
- role 名を文字列で各所に直書きし、権限変更時に漏れる
- middleware に業務判断を入れ、テストしにくくなる
- 例外の種類ごとにエラー JSON の形がばらばらになる
- CORS を緩めることで API の認可問題を解決したつもりになる
- health check が DB や外部 API に深く依存し、監視自体が不安定になる
- Minimal API の handler が巨大化し、Controller より読みにくくなる

## レビュー観点

- 認証方式、認可ポリシー、匿名許可の範囲が明示されているか
- 横断処理が middleware、filter、endpoint filter の適切な場所に置かれているか
- API の失敗が統一された `ProblemDetails` と trace id で返るか
- rate limit、CORS、health checks が本番環境の脅威と運用に合っているか
- DTO、handler、use case、domain の責務が混ざっていないか
- エンドポイント単位のテストと use case 単位のテストを分けやすいか
- 互換性を壊す変更、ステータスコード変更、エラーコード変更が意識されているか

## 記事一覧

- [Authentication / Authorization](authentication-authorization.md) - 認証と認可を分け、policy と claim を実務判断に落とし込むための入口です。
- [Middleware / Filters / Endpoint Filters](middleware-filters-endpoint-filters.md) - 横断処理の置き場所を、責務とテスト容易性から選ぶ観点です。
- [ProblemDetails とエラー応答](problemdetails-error-responses.md) - API の失敗を一貫した公開契約として設計する考え方です。
- [Rate Limit / CORS / Health Checks](rate-limit-cors-health-checks.md) - 防御、ブラウザー制約、運用監視を混同しないための整理です。
- [Minimal API / Controller の境界](minimal-api-controller-boundaries.md) - エンドポイント実装方式を、複雑さとチーム運用で選ぶ判断材料です。
- [レビュー checklist](checklist.md) - Part 12 の観点を PR レビューで確認できる形にまとめています。

---

[章別ノート一覧に戻る](../index.md)
