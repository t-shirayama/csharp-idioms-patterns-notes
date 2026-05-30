# raw SQL と injection 対策

## 概要

EF Core でも、複雑な集計、DB 固有機能、性能上の理由で raw SQL を使うことがあります。raw SQL は強力ですが、SQL injection、結果型の不一致、DB 依存、権限、監査の問題を持ち込みます。使う場合は「なぜ LINQ ではなく SQL か」と「入力をどう安全に扱うか」を明確にします。

## 判断基準

- LINQ では表現しにくい、または生成 SQL が明らかに不適切な場合に使う
- 値はパラメータとして渡す
- テーブル名、列名、並び順などパラメータ化できない識別子は whitelist で制限する
- raw SQL の結果型と列名をテストで確認する
- DB 製品依存の SQL は移植性より実務上の価値があるか判断する

## 使いどころ

window function、全文検索、複雑な集計、DB 固有の lock hint、既存 stored procedure 呼び出しなどで raw SQL が候補になります。ただし、単純な検索条件や並び替えまで raw SQL にすると、EF Core の型安全性や composability を失いやすくなります。

## 良い例

値は補間文字列を通してパラメータ化します。

```csharp
var orders = await db.Orders
    .FromSqlInterpolated($"""
        SELECT *
        FROM Orders
        WHERE CustomerId = {customerId}
        AND OrderedAt >= {from}
        """)
    .AsNoTracking()
    .ToListAsync(cancellationToken);
```

集計結果は専用型に投影します。

```csharp
var summaries = await db.Database
    .SqlQuery<OrderMonthlySummary>($"""
        SELECT CustomerId, COUNT(*) AS OrderCount, SUM(TotalAmount) AS TotalAmount
        FROM Orders
        WHERE OrderedAt >= {from}
        GROUP BY CustomerId
        """)
    .ToListAsync(cancellationToken);
```

## 避けたい書き方

ユーザー入力を文字列連結で埋め込むと SQL injection の危険があります。

```csharp
var sql = "SELECT * FROM Orders WHERE CustomerId = " + customerId;

var orders = await db.Orders
    .FromSqlRaw(sql)
    .ToListAsync(cancellationToken);
```

並び順や列名も注意が必要です。

```csharp
var sql = $"SELECT * FROM Orders ORDER BY {sortColumn}";
```

値ではなく識別子は通常のパラメータ化ができないため、別の防御が必要です。

## 改善例

値はパラメータ化し、識別子は whitelist で選びます。

```csharp
var orderBy = sortColumn switch
{
    "orderedAt" => "OrderedAt",
    "totalAmount" => "TotalAmount",
    _ => "OrderedAt"
};

var sql = $"""
    SELECT *
    FROM Orders
    WHERE CustomerId = @customerId
    ORDER BY {orderBy}
    """;

var orders = await db.Orders
    .FromSqlRaw(sql, new SqlParameter("@customerId", customerId))
    .AsNoTracking()
    .ToListAsync(cancellationToken);
```

可能なら、動的条件は LINQ に戻します。

```csharp
var query = db.Orders.AsNoTracking()
    .Where(order => order.CustomerId == customerId);

query = sortColumn switch
{
    "totalAmount" => query.OrderByDescending(order => order.TotalAmount),
    _ => query.OrderByDescending(order => order.OrderedAt)
};

var orders = await query.ToListAsync(cancellationToken);
```

## レビュー観点

- raw SQL を使う理由が説明されているか
- 値がパラメータ化されているか
- 識別子、並び順、SQL 断片にユーザー入力を直接使っていないか
- 結果型と SQL の列名が一致しているか
- 権限、監査ログ、実行計画、タイムアウトを確認しているか
- LINQ で十分な処理まで raw SQL に寄せていないか

<!-- TODO対応追記 -->

## TODO対応: ExecuteSqlInterpolated

> 対応元: P0 / `ExecuteSqlInterpolated`、`ExecuteUpdate/Delete` との使い分け、権限最小化、監査ログ、機密値ログ出力の注意を追加する。

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
