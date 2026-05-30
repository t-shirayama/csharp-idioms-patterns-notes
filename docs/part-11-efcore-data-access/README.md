# Part 11: EF Core / DB アクセス深掘り

EF Core は、SQL を書かずに DB を扱うための道具ではなく、C# の式を SQL、トランザクション、変更追跡、スキーマ変更へ変換する境界です。実務では、コードが短いかどうかよりも、発行されるクエリ、読み込む列と行数、整合性、同時更新、migration 運用、セキュリティを確認できることが重要です。

この Part では、EF Core を使うコードをレビューするときに迷いやすい判断を整理します。Repository pattern を使うかどうか、`Include` をどこまで許すか、`AsNoTracking()` を付けるべきか、raw SQL を安全に扱えているか、といった設計上の境界を扱います。

## このPartで身につくこと

- tracking / no tracking / projection を、読み取り、更新、性能の観点で選べる。
- `Include`、projection、N+1、split query を、クエリ形状としてレビューできる。
- トランザクション境界、楽観的同時実行制御、再試行の責務を整理できる。
- migration を「自動生成ファイル」ではなく、スキーマ変更の設計レビュー対象として扱える。
- EF Core と Repository pattern の相性を、抽象化の目的から判断できる。
- raw SQL、動的条件、SQL injection を安全に扱う基準を説明できる。
- DB アクセスのテストで、InMemory provider に頼りすぎない判断ができる。

## 読む順番

まず [tracking / no tracking / projection](tracking-no-tracking-projection.md) で、読み取りと更新の基本的なクエリ設計を押さえます。

次に [Include / クエリ形状 / N+1](include-query-shape-n-plus-one.md) を読み、ナビゲーションプロパティをどこまで読み込むか、SQL とデータ量の観点で確認します。

[トランザクションと同時実行制御](transactions-concurrency.md) では、複数更新、同時更新、リトライの扱いを整理します。その後、[migration とスキーマ変更](migrations-and-schema-changes.md) で、DB スキーマを安全に進化させる観点を確認します。

設計境界に迷う場合は [EF Core と Repository pattern](repository-pattern-with-efcore.md) を読み、最後に [raw SQL と injection 対策](raw-sql-and-injection.md) と [レビュー checklist](checklist.md) で、PR 上の確認項目に落とし込みます。

## 重要トピック

- 読み取り専用クエリは `AsNoTracking()` と projection を基本候補にする
- 更新する Entity は tracking の意味を理解して扱う
- `Include` は便利な読み込み指定ではなく、SQL と結果セットの形を変える判断として見る
- N+1 は小さいテストデータでは見えにくく、ログや SQL 確認が必要になる
- トランザクションは「メソッド全体」ではなく、整合性を守る最小単位で考える
- migration はロールバック、既存データ、ダウンタイム、インデックス作成時間まで含めて確認する
- raw SQL は補間文字列の見た目だけで安全性を判断しない

## 実務での使いどころ

一覧 API、検索画面、バッチ処理では、必要な列だけを projection し、ページングと並び順を明示します。Entity graph をそのまま読み込むと、レスポンスには不要な列、巨大な join、意図しない lazy loading が混ざりやすくなります。

更新処理では、DB 制約、同時更新、トランザクション、イベント発行、外部 I/O の境界を分けて考えます。DB の変更とメール送信や HTTP 呼び出しを同じトランザクション内に入れると、失敗時の扱いが難しくなります。

スキーマ変更では、migration をレビューし、既存データへの影響、nullable 変更、default 値、インデックス、長時間ロックの可能性を確認します。アプリケーションコードだけでは安全性を判断できません。

## よくある落とし穴

- 一覧表示で Entity を丸ごと取得し、必要な列より多く読む
- 読み取り専用なのに tracking したまま大量データを処理する
- `Include` を増やして問題を隠し、巨大な join や重複行を作る
- ループ内でナビゲーションを参照し、N+1 を発生させる
- migration を生成結果のまま適用し、既存データやロールバックを確認しない
- Repository が `IQueryable<T>` を無制限に漏らし、抽象化の意味がなくなる
- raw SQL の文字列連結で injection や壊れやすい動的 SQL を作る

## レビュー観点

- クエリは必要な列、必要な件数、明示的な並び順、ページングを持っているか
- `AsNoTracking()`、tracking、projection の選択理由が読み取れるか
- `Include` の数、join の形、split query の必要性、N+1 の可能性を確認しているか
- 更新処理のトランザクション境界と同時更新時の挙動が明確か
- migration に既存データ、NULL、default、index、rollback、デプロイ順序の考慮があるか
- Repository / Service / UseCase の境界が EF Core の機能を隠しすぎていないか
- raw SQL はパラメータ化され、識別子や並び順の動的入力を whitelist で制限しているか

## 記事一覧

- [tracking / no tracking / projection](tracking-no-tracking-projection.md) - 読み取り、更新、DTO 化、性能の観点で EF Core の基本クエリ形を選びます。
- [Include / クエリ形状 / N+1](include-query-shape-n-plus-one.md) - ナビゲーション読み込みを SQL とデータ量の観点でレビューします。
- [トランザクションと同時実行制御](transactions-concurrency.md) - 整合性を守る境界、楽観的同時実行制御、リトライを扱います。
- [migration とスキーマ変更](migrations-and-schema-changes.md) - スキーマ変更を安全に本番へ反映するための確認観点です。
- [EF Core と Repository pattern](repository-pattern-with-efcore.md) - Repository を使う場合、使わない場合、`IQueryable` の扱いを整理します。
- [raw SQL と injection 対策](raw-sql-and-injection.md) - EF Core で raw SQL を使う判断と、SQL injection を避ける実装を扱います。
- [レビュー checklist](checklist.md) - EF Core / DB アクセスの PR で確認する項目をまとめます。

---

[章別ノート一覧に戻る](../index.md)

<!-- TODO対応追記 -->

## TODO対応: multi-tenancy

> 対応元: P2 / multi-tenancy、soft delete、global query filter、tenant 漏れの注意を新規章候補として追加する。

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
