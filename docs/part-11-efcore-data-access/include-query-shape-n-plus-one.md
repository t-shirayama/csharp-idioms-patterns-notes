# Include / クエリ形状 / N+1

## 概要

`Include` は「関連データも取る」ための便利指定ですが、実際には SQL の join、結果セットの重複、読み込む行数、メモリ使用量を変えます。レビューでは C# の見た目だけでなく、発行される SQL とデータ量を確認します。

## 判断基準

- レスポンスや処理に必要な関連データだけを読み込む
- 一覧では Entity graph より DTO projection を優先する
- 複数 collection の `Include` は cartesian explosion に注意する
- ループ内でナビゲーション参照による追加クエリが出ないか確認する
- 必要に応じて `AsSplitQuery()`、projection、明示的な集計を検討する

## 使いどころ

詳細画面のように、一つの aggregate と関連データをまとめて扱う場合は `Include` が分かりやすいことがあります。一覧、検索、API レスポンスでは、必要な列と件数が明確になりやすい projection が向いています。

## 良い例

一覧では必要な形に直接 projection します。

```csharp
var customers = await db.Customers
    .AsNoTracking()
    .OrderBy(customer => customer.Name)
    .Select(customer => new CustomerListItemDto(
        customer.Id,
        customer.Name,
        customer.Orders.Count))
    .Take(100)
    .ToListAsync(cancellationToken);
```

詳細で collection を複数読む場合は、split query を検討します。

```csharp
var customer = await db.Customers
    .AsNoTracking()
    .Include(customer => customer.Orders)
    .Include(customer => customer.Contacts)
    .AsSplitQuery()
    .SingleAsync(customer => customer.Id == customerId, cancellationToken);
```

## 避けたい書き方

N+1 は、ループ内のナビゲーション参照や追加クエリで発生します。

```csharp
var customers = await db.Customers.ToListAsync(cancellationToken);

foreach (var customer in customers)
{
    Console.WriteLine(customer.Orders.Count);
}
```

lazy loading が有効な場合、このコードは顧客ごとに注文を読む可能性があります。小さい開発データでは気づきにくい問題です。

## 改善例

必要な集計をクエリ内に含めます。

```csharp
var summaries = await db.Customers
    .AsNoTracking()
    .Select(customer => new CustomerOrderSummary(
        customer.Id,
        customer.Name,
        customer.Orders.Count))
    .ToListAsync(cancellationToken);
```

関連データが本当に必要な詳細取得では、対象を一件に絞ってから読み込みます。

```csharp
var order = await db.Orders
    .AsNoTracking()
    .Include(order => order.Lines)
    .SingleAsync(order => order.Id == orderId, cancellationToken);
```

## レビュー観点

- `Include` で読み込む関連データが実際に使われているか
- 一覧 API で Entity graph を返そうとしていないか
- ループ内で DB クエリが増えないか
- 複数 collection の `Include` で結果セットが膨らまないか
- SQL ログや実行計画でクエリ形状を確認できるか
- projection、`AsSplitQuery()`、ページングの検討がされているか

<!-- TODO対応追記 -->

## TODO対応: filtered include

> 対応元: P1 / filtered include、split query の整合性・往復回数の trade-off、ページングと Include の組み合わせを追加する。

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
