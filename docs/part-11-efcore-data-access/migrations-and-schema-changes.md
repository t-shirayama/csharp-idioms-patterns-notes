# migration とスキーマ変更

## 概要

EF Core migration は、モデル差分から SQL を生成する便利な仕組みですが、生成された migration が本番データに安全とは限りません。既存データ、NULL、default、index、ロック時間、デプロイ順序、ロールバックを含めてレビューします。

## 判断基準

- 空の DB ではなく、既存データがある DB に適用できるか確認する
- nullable から non-null への変更はデータ補正と default を設計する
- index 追加はサイズ、ロック、重複、クエリ計画への影響を見る
- rename と drop/add の違いを確認し、意図しないデータ消失を避ける
- アプリケーションと DB のデプロイ順序に互換性があるか考える

## 使いどころ

新しい列の追加、列名変更、制約追加、index 追加、テーブル分割、データ移行のように、スキーマが本番データに影響する変更で重要です。migration ファイルは自動生成物ではなく、レビュー対象の設計成果物として扱います。

## 良い例

既存データがある列を non-null にする場合は、段階的に変更します。

```csharp
migrationBuilder.AddColumn<string>(
    name: "DisplayName",
    table: "Users",
    type: "nvarchar(200)",
    maxLength: 200,
    nullable: true);

migrationBuilder.Sql("""
    UPDATE Users
    SET DisplayName = UserName
    WHERE DisplayName IS NULL
    """);

migrationBuilder.AlterColumn<string>(
    name: "DisplayName",
    table: "Users",
    type: "nvarchar(200)",
    maxLength: 200,
    nullable: false,
    oldClrType: typeof(string),
    oldType: "nvarchar(200)",
    oldNullable: true);
```

## 避けたい書き方

生成された migration を確認せず、意図しない drop/add を含めたまま適用するのは危険です。

```csharp
migrationBuilder.DropColumn(
    name: "UserName",
    table: "Users");

migrationBuilder.AddColumn<string>(
    name: "LoginName",
    table: "Users",
    type: "nvarchar(max)",
    nullable: false,
    defaultValue: "");
```

列名変更のつもりなら、既存データを失う可能性があります。

## 改善例

rename であることを明示します。

```csharp
migrationBuilder.RenameColumn(
    name: "UserName",
    table: "Users",
    newName: "LoginName");
```

大きなテーブルの index は、DB 製品ごとのオンライン作成、実行時間、ロールバック手順を確認します。

```csharp
migrationBuilder.CreateIndex(
    name: "IX_Orders_CustomerId_OrderedAt",
    table: "Orders",
    columns: new[] { "CustomerId", "OrderedAt" });
```

## レビュー観点

- migration の Up / Down が意図どおりか
- drop/add が rename や型変更の誤検出ではないか
- 既存データに対する補正 SQL が必要ないか
- non-null、unique、foreign key、check constraint の追加で失敗しないか
- index 追加で本番 DB のロックや時間が問題にならないか
- アプリケーションの旧版と新版が一時的に同じ DB を使っても壊れないか
- rollback できる変更か、できない場合の復旧手順があるか

<!-- TODO対応追記 -->

## TODO対応: expand-contract

> 対応元: P0 / expand-contract、後方互換 migration、idempotent script / migration bundle、本番適用手順、バックアップ・復旧を追加する。

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
