# range、index、tuple、deconstruction、匿名型

## 概要

range、index、tuple、deconstruction、匿名型は、局所的なデータ操作を短く書くための機能です。便利ですが、public API やドメインモデルの代替として使いすぎると、意味が名前に残りにくくなります。

これらは「近くを読めば分かる」ことが前提の構文です。便利な短縮記法で仕様上の概念を隠すと、レビュー時に値の意味や順序を追う負担が増えます。

## 判断基準

これらの機能は「近くに文脈があり、短く書くことで読みやすくなる」場面で使います。メソッドをまたぐ、public API になる、テストデータとして何度も使う、仕様用語として説明したい、という場合は record や専用クラスに名前を付けた方が安全です。

range と index は境界条件を短く書けますが、入力が空や少数件のときに例外になるかを確認します。tuple は要素名を付けても型としての意味は弱いため、値の順序依存が強くなる点に注意します。

range は配列ではコピーを作り、`Span<T>` や `ReadOnlySpan<T>` ではビューとして扱えるなど、対象の型によってコストが変わります。通常の業務コードでは過剰に気にしすぎる必要はありませんが、大きな配列や高頻度処理では意図したコストかを確認します。

deconstruction は、値の意味が左辺の名前で十分に読める場合に向きます。未使用値が多い、順番を覚えないと読めない、同じ形の tuple が複数箇所に出る場合は、名前付きの型へ寄せます。

## 使いどころ

- 配列や `Span<T>` の末尾参照、部分範囲取得では `^` と `..` が読みやすい。
- メソッド内部の一時的な複数値には tuple が向いている。
- LINQ の投影やグルーピングでは匿名型が自然に使える。
- deconstruction は、戻り値をすぐ近くで分解して使い切る場合に向く。
- 匿名型は、クエリ内部で一時的に列を増やす用途に留める。

```csharp
var recentOrders = orders.Length >= 3
    ? orders[^3..]
    : orders[..];

var (subtotal, tax) = CalculateAmounts(recentOrders);
var total = subtotal + tax;
```

件数不足があり得る場合は、range/index の前に扱いを決めます。

```csharp
public Order[] TakeRecentOrders(Order[] orders)
{
    return orders.Length <= 3
        ? orders
        : orders[^3..];
}
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

deconstruction は、分解した値をすぐ使うと読みやすくなります。遠くまで変数を持ち回るなら、分解しない方が意味が残ることもあります。

```csharp
var (subtotal, tax) = CalculateAmounts(order);

return new InvoiceAmounts(
    Subtotal: subtotal,
    Tax: tax,
    Total: subtotal + tax,
    IsDiscounted: order.HasDiscount);
```

## 避けたい書き方

- public メソッドの戻り値に大きな tuple を使い、各値の意味を呼び出し側に覚えさせる。
- range の境界を確認せず、短い入力で例外になる。
- 匿名型を境界の外へ持ち出そうとして、型の責務が曖昧になる。
- tuple の要素順に業務上の意味を持たせ、呼び出し側に順番を覚えさせる。
- range によるコピーコストが大きい箇所で、意図せず毎回部分配列を作る。

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

匿名型はメソッド境界を越えられないため、無理に `object` や `dynamic` へ逃がすと型安全性を失います。

```csharp
// 避けたい例: 匿名型を外へ出すために型情報を捨てている
public object BuildSummary(Order order)
{
    return new { order.Id, order.TotalAmount };
}
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

同じ形の tuple や匿名型を繰り返すなら、仕様上の概念として名前を付けます。型名が付くと、テスト名やレビューコメントでも同じ語彙を使えます。

```csharp
public sealed record CustomerOrderSummary(
    CustomerId CustomerId,
    decimal TotalAmount);
```

## レビュー観点

- tuple の要素名だけで十分か、専用の record を作るべきか。
- range や index の境界条件が、空、1件、少数件のテストで確認されているか。
- deconstruction によって、値の順序依存が読みにくくなっていないか。
- 匿名型がクエリ内部の一時表現に収まり、外部契約になっていないか。
- `^1` や `..` が読み手にとって自然な文脈か、明示的な変数名を置くべきか。
- tuple の値を何度も受け渡しているなら、仕様上の概念として名前を付けるべきか。
- range がコピーを作る場面で、入力サイズや呼び出し頻度に対して妥当か。
- deconstruction 後の変数名が、元の値の意味を失っていないか。

## テスト観点

- range/index は、空、1件、必要件数ちょうど、境界超えを確認する。
- `^1` のような末尾参照は、空入力を例外にするか null にするかを仕様として固定する。
- tuple の戻り値は、要素順を間違えてもテストで気付ける期待値にする。
- deconstruction を使う処理は、未使用値や順序依存が仕様変更時に壊れないか確認する。
- 匿名型を record へ移した場合、型名がテスト名や期待値の読みやすさに貢献しているかを見る。

---

[Part README に戻る](README.md)


