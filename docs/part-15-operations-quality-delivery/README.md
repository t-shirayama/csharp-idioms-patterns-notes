# Part 15: 運用・品質・配布補足

実務の C# アプリケーションは、コードが正しく動くだけでは足りません。本番で何が起きているかを観測できること、壊れたときに早く切り分けられること、CI で品質を落とさないこと、安全に設定や secret を扱えること、ライブラリを破壊的変更なしに配布できることが重要です。

この Part では、ログだけでは追いきれない metrics、tracing、OpenTelemetry、health checks、CI 品質ゲート、NuGet とバージョニング、secret 運用を扱います。目的は「運用基盤を全部作ること」ではなく、アプリケーションコードを書く人がレビューで見落としやすい運用品質の判断基準を持つことです。

## このPartで身につくこと

- logs、metrics、traces の役割を分け、障害調査に必要な情報を設計できる。
- OpenTelemetry を、導入しただけで満足せず、span、tag、cardinality、sampling の観点で確認できる。
- readiness、liveness、startup checks を分け、health check を監視やデプロイ制御に使える。
- CI で analyzer、formatter、test、coverage、package 検証をどこまで品質ゲートにするか判断できる。
- NuGet パッケージの SemVer、public API、互換性、依存関係、配布メタデータをレビューできる。
- 設定と secret を、環境変数、Key Vault、User Secrets、Options validation と組み合わせて安全に扱える。

## 読む順番

まず [metrics / tracing / OpenTelemetry](metrics-tracing-opentelemetry.md) を読み、ログだけでは足りない観測性の土台を押さえます。

次に [health checks、readiness、liveness](health-checks-readiness-liveness.md) で、監視とデプロイ制御に使うヘルスチェックの分け方を確認します。

その後、[CI、analyzers、formatters、tests](ci-analyzers-formatters-tests.md) で、品質ゲートをチームの開発速度に合わせて設計する観点を整理します。

ライブラリを作る場合は [NuGet、versioning、library design](nuget-versioning-library-design.md) を読み、アプリケーション運用では [configuration and secret operations](configuration-and-secret-operations.md) を合わせて確認します。

最後に [レビュー checklist](checklist.md) で、PR やリリース前確認に使う観点へ落とし込みます。

## 重要トピック

- logs は個別イベント、metrics は傾向、traces は処理の流れを見るために使い分ける。
- OpenTelemetry の tag は便利だが、ユーザー ID やリクエスト ID など高 cardinality な値を metric label に入れない。
- health check は「プロセスが生きているか」と「リクエストを受けてよいか」を分ける。
- CI は警告を全部失敗にする前に、既存負債、段階導入、例外ルール、ローカル再現性を確認する。
- NuGet はコードだけでなく、互換性、依存 version 範囲、README、license、source link、symbols も配布物に含めて考える。
- secret は設定ファイルに置かず、取得経路、ローテーション、ログ漏えい、権限境界まで含めて設計する。

## 実務での使いどころ

本番障害の調査では、例外ログだけでは「どの依存先が遅いか」「どの endpoint の失敗率が上がったか」「あるリクエストがどの処理で詰まったか」までは分かりません。metrics と traces を設計しておくと、調査の入口を作れます。

デプロイやインフラ連携では、health check の設計が重要です。DB が落ちているときに liveness を失敗させると、コンテナが再起動を繰り返し、かえって原因を見えにくくすることがあります。

ライブラリや共通基盤を配布する場合は、NuGet の versioning と public API 互換性が利用者の更新コストを左右します。アプリケーションの場合も、CI と secret 運用を整えることで、変更の安全性とリリースの再現性が上がります。

## よくある落とし穴

- ログを増やしただけで観測性を上げたつもりになり、metrics や traces がない。
- metric label に request path、user id、raw query などを入れて cardinality を爆発させる。
- liveness check で DB や外部 API を叩き、依存先障害時に自アプリまで再起動させる。
- CI の analyzer 警告を一気に error 化し、既存負債で開発が止まる。
- NuGet の patch 更新で public API を壊し、利用者側に予期しないビルドエラーを出す。
- secret をログ、例外、設定ダンプ、CI の出力に出してしまう。

## レビュー観点

- 重要なユースケースに、ログ、metric、trace のどれで追うかが設計されているか。
- OpenTelemetry の span 名、tag、error 設定、sampling が後から調査しやすい単位になっているか。
- readiness と liveness が分かれ、依存先障害時の挙動が意図どおりか。
- CI で formatter、analyzer、test、coverage、package 検証のどれを必須にするか説明できるか。
- NuGet の version 更新が SemVer と public API 互換性に合っているか。
- secret の保存場所、注入方法、ローテーション、ログ漏えい防止が確認されているか。

## 記事一覧

- [metrics / tracing / OpenTelemetry](metrics-tracing-opentelemetry.md) - logs、metrics、traces を分け、調査に使える観測性を設計する観点です。
- [health checks、readiness、liveness](health-checks-readiness-liveness.md) - ヘルスチェックを監視、デプロイ、再起動制御に安全に使うための判断基準です。
- [CI、analyzers、formatters、tests](ci-analyzers-formatters-tests.md) - 品質ゲートを開発速度と既存負債に合わせて段階導入する考え方です。
- [NuGet、versioning、library design](nuget-versioning-library-design.md) - ライブラリ配布、互換性、SemVer、依存関係をレビューするための入口です。
- [configuration and secret operations](configuration-and-secret-operations.md) - 設定と secret を本番運用で安全に扱うための設計観点です。
- [レビュー checklist](checklist.md) - Part 15 の観点を PR、リリース、運用レビューで確認するための一覧です。

---

[章別ノート一覧に戻る](../index.md)

<!-- TODO対応追記 -->

## TODO対応: Part 14 との接続として「性能改善後に何を metric/trace/aler…

> 対応元: P2 / Part 14 との接続として「性能改善後に何を metric/trace/alert で見るか」を追加する。

### 使いどころ

この観点は、単なる構文選択ではなく、境界、運用、テスト、レビューで後から判断できる形にしておくために扱います。新規開発では最初から方針を決め、既存コードでは事故が起きやすい境界から段階的に直します。

### 避けたい例

```csharp
// 例: 境界の責務や失敗時の扱いを呼び出し側の暗黙知にしている。
var value = input.ToString();
Save(value);
```

この書き方は小さなサンプルでは動きますが、値の追加、設定変更、外部入力、再試行、キャンセル、ログ出力、テスト実行時に意図しない挙動を隠しやすくなります。

### 改善例

```csharp
public static Result<ValidatedValue> Create(string? input)
{
    if (string.IsNullOrWhiteSpace(input))
    {
        return Result.Invalid("入力が空です。");
    }

    return Result.Success(new ValidatedValue(input.Trim()));
}
```

境界で正規化し、失敗理由を型または戻り値で表すと、レビューとテストで確認する場所が明確になります。例外にするか `Result` にするかは、呼び出し側が復旧できるか、業務エラーとして利用者に返すかで判断します。

### レビュー観点

- 追加・変更時に未対応ケースを隠さず検知できるか。
- 外部入力、DB、JSON、設定、環境差分などの境界で正規化と検証を行っているか。
- 失敗、キャンセル、timeout、再試行、部分成功の扱いがテスト名から読み取れるか。
- ログや例外に機密値、巨大データ、内部実装を出しすぎていないか。
- パフォーマンス改善は計測に基づき、読みやすさを壊すだけの過剰最適化になっていないか。
