# if / switch / pattern matching

## 概要

条件分岐は、業務ルールが最も露出しやすい場所です。if、switch expression、type pattern、property pattern、relational pattern、list pattern は、条件の形に合わせて選びます。

## 判断基準

条件の数が少なく、副作用を伴う処理なら `if` が自然です。入力値から別の値へ変換するだけなら `switch expression` が読みやすく、戻り値の網羅性も見えます。オブジェクトの複数プロパティを見るなら property pattern を使うと、null チェックと条件をまとめられます。

一方で、pattern matching は業務語彙の名前を置き換えるものではありません。条件が重要なルールを表すなら、`IsPriorityReviewRequired` のようなメソッド名に抽出した方がレビューしやすくなります。

## 使いどころ

- 単純な真偽判定は if で十分に書く。
- 値から値へ変換する分岐は switch expression にすると戻り値の抜け漏れを見つけやすい。
- オブジェクトの状態を見る条件は property pattern でまとめる。

```csharp
public static string GetReviewLabel(Order order) => order switch
{
    { Status: OrderStatus.Canceled } => "確認不要",
    { TotalAmount: >= 100_000m, Customer.IsPreferred: true } => "優先レビュー",
    { TotalAmount: >= 100_000m } => "上長レビュー",
    _ => "通常レビュー"
};
```

## 良い例

条件に業務上の名前を付けると、テスト名やレビューコメントでも同じ語彙を使えます。switch expression は分岐の結果が値として閉じている場面に使います。

```csharp
private static bool RequiresManagerApproval(Order order)
{
    return order is
    {
        Status: OrderStatus.Submitted,
        TotalAmount: >= 100_000m
    };
}

var nextStatus = RequiresManagerApproval(order)
    ? OrderStatus.WaitingForApproval
    : OrderStatus.Approved;
```

## 避けたい書き方

- 条件式を1行に詰め込み、業務上の意味が名前として残っていない。
- switch の default で想定外ケースを握りつぶす。
- pattern matching を使いすぎて、単純な if より読みにくくなる。

```csharp
if (order.Status != OrderStatus.Canceled && order.TotalAmount >= 100000 && order.Customer != null && order.Customer.IsPreferred)
{
    label = "優先レビュー";
}
```

pattern を入れ子にしすぎると、条件の優先順位が読み手に伝わりにくくなります。特に条件の順序で結果が変わる場合は、限定的な条件を先に置き、テストで境界を固定します。

```csharp
// 避けたい例: 条件の意味より構文の複雑さが目立つ
var fee = order.Customer is { Rank: CustomerRank.Gold, Address.Country: "JP" }
    && order.Items is [_, _, ..]
    ? 0m
    : 800m;
```

## 改善例

複雑な条件は小さな predicate に分け、分岐本体では判断の流れを見せます。条件をテストできる単位にすることで、仕様変更時の差分も小さくなります。

```csharp
var fee = IsFreeShippingTarget(order)
    ? 0m
    : ShippingFee.Standard;

static bool IsFreeShippingTarget(Order order)
{
    return order.Customer.Rank == CustomerRank.Gold
        && order.Customer.Address.Country == "JP"
        && order.Items.Count >= 2;
}
```

## レビュー観点

- 複雑な条件に、`IsHighValueOrder` など業務語彙の名前を付けられないか。
- switch expression の `_` が本当に安全な既定値か、例外にすべき想定外か。
- 条件の順序に意味がある場合、より限定的な条件が先に来ているか。
- list pattern は入力サイズや境界条件をテストで押さえているか。
- null チェック、型チェック、業務条件が混ざりすぎていないか。
- 分岐に副作用がある場合、switch expression で無理に短くしていないか。

---

[Part README に戻る](README.md)


