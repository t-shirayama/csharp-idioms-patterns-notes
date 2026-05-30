# Part 12: ASP.NET Core / Web API レビュー checklist

このチェックリストは、ASP.NET Core の Web API をレビューするときに、認証、認可、横断処理、エラー応答、公開契約、運用観点を抜け漏れなく確認するためのものです。

すべての項目を毎回満たすための表ではありません。変更された API が「誰に何を許し、失敗時に何を返し、本番でどう観測されるか」を具体的に確認するために使います。

## Authentication / Authorization

- [ ] 認証方式が API の利用者に合っている。cookie、JWT bearer、mTLS、API key などの選択理由が説明できる。
- [ ] authentication と authorization が混ざっていない。ログイン済みであることと、操作できることを別に確認している。
- [ ] 匿名アクセスを許可する endpoint が明示され、既定で公開される状態になっていない。
- [ ] role、claim、scope、policy の使い分けが整理され、文字列の直書きが散らばっていない。
- [ ] テナント境界、所有者チェック、管理者権限など、データ単位の認可が endpoint か use case で確認されている。
- [ ] 401 と 403 の使い分けが適切で、認証失敗と権限不足を混同していない。
- [ ] 認可失敗時に、存在確認や機密情報がレスポンスから漏れない。

## Middleware / Filters / Endpoint Filters

- [ ] リクエスト全体に効く処理は middleware、MVC 固有の処理は filter、Minimal API の endpoint 境界は endpoint filter に置いている。
- [ ] 認可、例外変換、ログ、相関 ID、メトリクスなどの横断処理が handler や controller に重複していない。
- [ ] middleware の順序が正しい。routing、authentication、authorization、exception handling の順序を確認している。
- [ ] filter や endpoint filter に業務ロジックを入れすぎず、use case のテスト容易性を保っている。
- [ ] 短絡する処理がある場合、ログ、メトリクス、エラー応答の形が欠けていない。
- [ ] request body を複数回読む処理が、性能やストリーム位置に悪影響を出していない。

## ProblemDetails / Error Responses

- [ ] 失敗時のレスポンス形式が API 全体で統一されている。
- [ ] `status`、`title`、`detail`、`type`、`instance` の役割が混ざっていない。
- [ ] 利用者が修正できるエラーと、システム側で調査するエラーを区別している。
- [ ] trace id や correlation id が返り、ログと照合できる。
- [ ] 例外メッセージ、stack trace、SQL、内部 URL、認証情報がレスポンスに露出していない。
- [ ] model validation、認証失敗、認可失敗、404、409、429、500 の扱いが一貫している。
- [ ] エラーコードや `type` URI が、クライアントの分岐に耐える安定した契約になっている。

## Rate Limit / CORS / Health Checks

- [ ] rate limit の単位が IP、ユーザー、テナント、API key などの脅威モデルに合っている。
- [ ] 429 応答にクライアントが再試行を判断できる情報がある。
- [ ] CORS はブラウザー向けの制約として扱い、API の認証や認可の代わりにしていない。
- [ ] `AllowAnyOrigin`、`AllowAnyHeader`、`AllowAnyMethod`、credentials の組み合わせが本番で安全か確認している。
- [ ] liveness と readiness が分かれており、再起動判断とトラフィック受け入れ判断を混同していない。
- [ ] health check が重すぎず、外部依存の一時障害で監視が過剰に不安定にならない。
- [ ] health endpoint が機密情報や依存先の詳細を公開しすぎていない。

## Minimal API / Controller Boundaries

- [ ] Minimal API と Controller の選択が、規模、複雑さ、チームの保守性に合っている。
- [ ] endpoint handler に validation、mapping、認可、業務処理、永続化が詰め込まれていない。
- [ ] request DTO、response DTO、domain model、EF entity の境界が分かれている。
- [ ] OpenAPI に出る契約と実際の status code、response body が一致している。
- [ ] route group、tags、versioning、authorization がまとまり単位で設定されている。
- [ ] endpoint 単体テスト、use case 単体テスト、統合テストの切り分けがしやすい。

## レビューで止めて確認したい兆候

- [ ] `RequireAuthorization()` がない endpoint が増えているが、匿名公開の理由が書かれていない。
- [ ] `User.IsInRole("Admin")` のような文字列認可が複数箇所に散らばっている。
- [ ] handler や controller の中で try/catch とエラー JSON 組み立てを毎回書いている。
- [ ] CORS を緩める変更で、実際は認可やフロントエンド設定の問題を隠している。
- [ ] health check に高コストな処理や外部 API 呼び出しが増えている。
- [ ] Minimal API の lambda が長くなり、読み替えやテストが難しくなっている。

---

[Part README に戻る](README.md)
