# if / switch / pattern matching

## 概要

条件分岐は、業務ルールが最も露出しやすい場所です。if、switch expression、type pattern、property pattern、relational pattern、list pattern は、条件の形に合わせて選びます。

実務で重要なのは、新しい構文を使うこと自体ではなく、条件の意味と優先順位が読み手に残ることです。条件が増えるほど、分岐は単なる制御構文ではなく仕様の表現になります。

## 判断基準

条件の数が少なく、副作用を伴う処理なら `if` が自然です。入力値から別の値へ変換するだけなら `switch expression` が読みやすく、戻り値の網羅性も見えます。オブジェクトの複数プロパティを見るなら property pattern を使うと、null チェックと条件をまとめられます。

一方で、pattern matching は業務語彙の名前を置き換えるものではありません。条件が重要なルールを表すなら、`IsPriorityReviewRequired` のようなメソッド名に抽出した方がレビューしやすくなります。

`switch expression` は値を返す分岐に向いています。ログ出力、DB 更新、通知送信などの副作用を混ぜると、式の短さに対して意図が重くなります。副作用があるなら `if` や `switch statement` で手順として見せる方が安全です。

pattern の順序にも注意します。より広い条件を先に置くと、後続の限定的な条件に到達しません。`_` は便利ですが、想定外値を通常扱いに落とすと、値追加時の不具合を隠します。

## 使いどころ

- 単純な真偽判定は if で十分に書く。
- 値から値へ変換する分岐は switch expression にすると戻り値の抜け漏れを見つけやすい。
- オブジェクトの状態を見る条件は property pattern でまとめる。
- 条件の意味がレビューで議論されるなら、predicate メソッドに名前を付ける。
- list pattern は、先頭や末尾の形が仕様として意味を持つ場合に使う。

```csharp
public static string GetReviewLabel(Order order) => order switch
{
    { Status: OrderStatus.Canceled } => "確認不要",
    { TotalAmount: >= 100_000m, Customer.IsPreferred: true } => "優先レビュー",
    { TotalAmount: >= 100_000m } => "上長レビュー",
    _ => "通常レビュー"
};
```

値の変換で例外にすべき未対応値は、`_` で握りつぶさず明示します。

```csharp
public static string ToLabel(OrderStatus status) => status switch
{
    OrderStatus.Draft => "下書き",
    OrderStatus.Submitted => "申請済み",
    OrderStatus.Canceled => "取消",
    _ => throw new ArgumentOutOfRangeException(nameof(status), status, null)
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

複数条件の優先順位が仕様で決まっている場合は、限定的な条件から並べます。並び順そのものが仕様なので、テスト名でも表現します。

```csharp
public static DiscountKind GetDiscount(Order order) => order switch
{
    { Customer.Rank: CustomerRank.Gold, TotalAmount: >= 50_000m } => DiscountKind.GoldLargeOrder,
    { Customer.Rank: CustomerRank.Gold } => DiscountKind.Gold,
    { TotalAmount: >= 50_000m } => DiscountKind.LargeOrder,
    _ => DiscountKind.None
};
```

## 避けたい書き方

- 条件式を1行に詰め込み、業務上の意味が名前として残っていない。
- switch の default で想定外ケースを握りつぶす。
- pattern matching を使いすぎて、単純な if より読みにくくなる。
- 副作用を伴う処理を、短さのために switch expression や三項演算子へ押し込む。
- null チェック、型チェック、権限判定、状態遷移をひとつの式に混ぜる。

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

副作用がある処理では、式に閉じ込めるより手順として書いた方がレビューしやすくなります。

```csharp
// 避けたい例: 分岐の結果と副作用が混ざっている
_ = order.Status switch
{
    OrderStatus.Submitted => notification.Send(order.Id),
    OrderStatus.Canceled => audit.LogCancel(order.Id),
    _ => Task.CompletedTask
};
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

副作用がある場合は、分岐ごとの手順を明示します。`switch statement` は古い書き方というより、命令的な処理を見せたい場面で今も有効です。

```csharp
switch (order.Status)
{
    case OrderStatus.Submitted:
        await notification.SendSubmittedAsync(order.Id);
        break;
    case OrderStatus.Canceled:
        await audit.LogCanceledAsync(order.Id);
        break;
}
```

## レビュー観点

- 複雑な条件に、`IsHighValueOrder` など業務語彙の名前を付けられないか。
- switch expression の `_` が本当に安全な既定値か、例外にすべき想定外か。
- 条件の順序に意味がある場合、より限定的な条件が先に来ているか。
- list pattern は入力サイズや境界条件をテストで押さえているか。
- null チェック、型チェック、業務条件が混ざりすぎていないか。
- 分岐に副作用がある場合、switch expression で無理に短くしていないか。
- property pattern により、単純な条件がかえって読みにくくなっていないか。
- 新しい状態や種別が増えたとき、未対応に気付ける構造になっているか。

## テスト観点

- 条件の境界値を、ちょうど未満、ちょうど、超過で確認する。
- 優先順位がある分岐は、複数条件に同時一致するケースを必ず置く。
- `_` や default に落ちるケースは、想定内の既定値か、想定外の例外かをテストで固定する。
- predicate に切り出した条件は、分岐本体とは別に業務名が分かるテスト名を付ける。
- list pattern は、空、1件、必要件数ちょうど、余分にあるケースを確認する。

---

[Part README に戻る](README.md)


