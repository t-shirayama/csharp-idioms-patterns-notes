# range、index、tuple、deconstruction、匿名型

## 概要

range、index、tuple、deconstruction、匿名型は、局所的なデータ操作を短く書くための機能です。便利ですが、public API やドメインモデルの代替として使いすぎると、意味が名前に残りにくくなります。

## 判断基準

これらの機能は「近くに文脈があり、短く書くことで読みやすくなる」場面で使います。メソッドをまたぐ、public API になる、テストデータとして何度も使う、仕様用語として説明したい、という場合は record や専用クラスに名前を付けた方が安全です。

range と index は境界条件を短く書けますが、入力が空や少数件のときに例外になるかを確認します。tuple は要素名を付けても型としての意味は弱いため、値の順序依存が強くなる点に注意します。

## 使いどころ

- 配列やリストの末尾参照、部分範囲取得では `^` と `..` が読みやすい。
- メソッド内部の一時的な複数値には tuple が向いている。
- LINQ の投影やグルーピングでは匿名型が自然に使える。

```csharp
var recentOrders = orders.Length >= 3
    ? orders[^3..]
    : orders[..];

var (subtotal, tax) = CalculateAmounts(recentOrders);
var total = subtotal + tax;
```

## 良い例

tuple は private メソッドやローカル関数で、すぐ近くの呼び出し側が意味を読める範囲に留めます。匿名型は LINQ の途中結果として使うと、不要な小型 DTO を増やさずに済みます。

```csharp
var summaries = orders
    .GroupBy(order => order.CustomerId)
    .Select(group => new
    {
        CustomerId = group.Key,
        Total = group.Sum(order => order.TotalAmount)
    });
```

```csharp
private static (decimal Subtotal, decimal Tax) CalculateAmounts(Order order)
{
    var subtotal = order.Items.Sum(item => item.Amount);
    return (subtotal, subtotal * 0.1m);
}
```

## 避けたい書き方

- public メソッドの戻り値に大きな tuple を使い、各値の意味を呼び出し側に覚えさせる。
- range の境界を確認せず、短い入力で例外になる。
- 匿名型を境界の外へ持ち出そうとして、型の責務が曖昧になる。

```csharp
public (decimal, decimal, decimal, bool) Calculate(Order order)
{
    return (order.Subtotal, order.Tax, order.Total, order.IsDiscounted);
}
```

```csharp
// 避けたい例: 入力が空だと例外になる
var lastOrder = orders[^1];
```

## 改善例

public API では値に名前を付けます。range や index を使う前に、空や件数不足をどう扱うかを明示します。

```csharp
public sealed record InvoiceAmounts(
    decimal Subtotal,
    decimal Tax,
    decimal Total,
    bool IsDiscounted);

public Order? GetLastOrder(IReadOnlyList<Order> orders)
{
    return orders.Count == 0 ? null : orders[^1];
}
```

## レビュー観点

- tuple の要素名だけで十分か、専用の record を作るべきか。
- range や index の境界条件が、空、1件、少数件のテストで確認されているか。
- deconstruction によって、値の順序依存が読みにくくなっていないか。
- 匿名型がクエリ内部の一時表現に収まり、外部契約になっていないか。
- `^1` や `..` が読み手にとって自然な文脈か、明示的な変数名を置くべきか。
- tuple の値を何度も受け渡しているなら、仕様上の概念として名前を付けるべきか。

---

[Part README に戻る](README.md)


