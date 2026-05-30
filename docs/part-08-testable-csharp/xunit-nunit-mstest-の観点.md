# xUnit / NUnit / MSTest の観点

## 概要

xUnit、NUnit、MSTest はどれも C# の代表的なテストフレームワークです。実務では機能差よりも、チームの既存資産、CI 連携、テスト名の読みやすさ、fixture の扱いが重要です。どのフレームワークでも arrange / act / assert を分けると、失敗時に何を検証していたかが読み取りやすくなります。

## 使いどころ

- xUnit: .NET の OSS や ASP.NET Core 周辺で採用例が多く、constructor と `IDisposable` による準備・後始末に寄せる。
- NUnit: `TestCase` などの属性が豊富で、パラメータ化テストを書きやすい。
- MSTest: Visual Studio や Microsoft 系の既存プロジェクトとの相性を重視する場面。

```csharp
[Theory]
[InlineData(1000, true, 900)]
[InlineData(1000, false, 1000)]
public void Calculate_applies_discount_for_preferred_customer(
    decimal subtotal,
    bool isPreferredCustomer,
    decimal expected)
{
    var calculator = new DiscountCalculator();

    var actual = calculator.Calculate(subtotal, isPreferredCustomer);

    Assert.Equal(expected, actual);
}
```

## フレームワーク選定の観点

新規プロジェクトでは、好き嫌いだけでなく運用を含めて選びます。

- 既存プロジェクトや社内テンプレートと揃っているか。
- CI の test logger、coverage、並列実行の設定が整っているか。
- parameterized test を読みやすく書けるか。
- fixture の共有範囲をチームで理解しやすいか。
- assertion library や mocking library との組み合わせが自然か。

既存プロジェクトでは、フレームワークを混在させるより、今あるものに合わせる方が保守しやすいです。移行するなら、テスト命名、fixture、CI 出力の差分まで計画します。

## テスト名と構造

テスト名は、失敗したときに仕様を思い出せる粒度にします。チームで形式を統一できていれば、日本語名でも英語名でも構いません。

```csharp
[Fact]
public void CanSubmit_returns_false_when_order_has_no_items()
{
    var order = new OrderBuilder()
        .WithoutItems()
        .Build();

    var actual = new OrderPolicy().CanSubmit(order);

    Assert.False(actual);
}
```

arrange が長い場合は、テストデータビルダーを使い、テストで重要な値だけ見えるようにします。setup に前提を隠しすぎると、テスト本体だけではなぜ失敗するか分からなくなります。

## fixture の扱い

fixture は、作成コストが高い依存を共有したいときに使います。ただし、共有状態は順序依存や並列実行の不安定さを生みやすいため、テストごとに初期化できるものは初期化します。

```csharp
public sealed class CalculatorTests
{
    private readonly DiscountCalculator calculator = new();

    [Fact]
    public void Calculate_does_not_discount_regular_customer()
    {
        var actual = calculator.Calculate(1000m, false);

        Assert.Equal(1000m, actual);
    }
}
```

状態を持たない対象ならフィールド共有でも問題になりにくいですが、`List<T>`、in-memory DB、fake clock のように状態が変わるものはテストごとに作る方が安全です。

## 避けたい書き方

- 1 つのテストで複数の仕様をまとめて検証し、失敗理由が分かりにくくなる。
- setup / teardown に重要な前提を隠し、テスト本体だけでは条件が読めない。
- パラメータ化テストに詰め込みすぎて、個別ケースの意図が失われる。
- テスト名が `Test1` や `Success` だけで、条件と期待結果が分からない。
- フレームワーク固有機能を使いすぎて、チーム内で読める人が限られる。

```csharp
// 避けたい例: 何の仕様が壊れたのか失敗名から分からない。
[Fact]
public void Test()
{
    Assert.True(service.Execute());
}
```

## レビュー観点

- テスト名から条件、操作、期待結果が推測できるか。
- arrange / act / assert の境界が読み取れるか。
- fixture がテスト間で状態を共有し、順序依存を生んでいないか。
- CI で並列実行しても安定するテストになっているか。
- パラメータ化テストの各ケースが、仕様上の意味を持っているか。
- 失敗時の assertion message や差分表示が、原因調査に十分か。

---

[Part README に戻る](README.md)


