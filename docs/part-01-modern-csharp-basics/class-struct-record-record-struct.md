# class / struct / record / record struct

## 概要

`class`、`struct`、`record`、`record struct` は、単なる好みではなく、同一性 (identity)、等価性 (equality)、変更可能性、コピーコストで選びます。

- `class` は参照同一性を持つオブジェクトに向く
- `struct` は小さく、値として扱う型に向く
- `record` は値ベースの等価性を持つ参照型に向く
- `record struct` は値型として等価性を持たせたい小さなデータに向く

## 判断基準

最初に考えるのは「同じ値なら同じものか」「同じ ID を持つ同一個体か」です。住所、金額、期間のように値の組み合わせが意味を持つなら `record` や `record struct` が候補になります。注文、顧客、セッションのように時間とともに状態が変わり、履歴やライフサイクルを持つなら `class` を優先します。

`record` は参照型ですが、等価性は値ベースです。キャッシュキー、重複排除、テストの期待値比較では便利ですが、Entity のように「同じ ID なら同じもの」「同じ値でも別インスタンスとして扱う」設計とは衝突することがあります。`record` を選ぶときは、自動生成される `Equals`、`GetHashCode`、`ToString`、`with` が公開 API の一部になると考えます。

`struct` は「値型にすると速そう」という理由だけで選ばない方が安全です。サイズが大きい、可変である、コレクションに入れて頻繁にコピーされる、interface 経由で扱う、という条件が重なると、かえって分かりにくいバグや余計なコストを生みます。

`record struct` は、座標、期間、金額のような小さな値に等価性を持たせたいときに向きます。C# 10 以降では `readonly record struct` にすると、コピー先で変更される不安を減らせます。ただし、値型なので `default` で不正な状態が作れる点は残ります。

## 使いどころ

Entity のように ID とライフサイクルを持つものは `class` が自然です。DTO や Value Object のように値の組み合わせが意味を持つものは `record` を検討します。

```csharp
public sealed class Customer
{
    public required Guid Id { get; init; }
    public required string Name { get; set; }
}

public sealed record EmailAddress(string Value);
```

小さな座標や範囲のように大量に扱う値は `readonly record struct` も候補になります。

```csharp
public readonly record struct Money(decimal Amount, string Currency);
```

`record` の positional syntax は短く書けますが、検証や将来のプロパティ追加が必要な型では通常のプロパティ形式の方が読みやすいことがあります。短さよりも、不変条件をどこで守るかを優先します。

## 良い例

Entity は ID と振る舞いを中心にし、Value Object は値の検証をコンストラクタやファクトリに寄せます。`record` の `with` は「一部だけ変えた新しい値」を作る用途に合いますが、Entity の状態遷移を表す道具として乱用しない方が読みやすくなります。

```csharp
public sealed record DateRange
{
    public DateRange(DateOnly start, DateOnly end)
    {
        if (end < start)
        {
            throw new ArgumentException("終了日は開始日以降にしてください。");
        }

        Start = start;
        End = end;
    }

    public DateOnly Start { get; }
    public DateOnly End { get; }
}
```

`with` は DTO や検索条件のように、不変条件が単純なデータのコピーに向きます。

```csharp
public sealed record SearchCondition(string Keyword, int Page);

var nextPage = currentCondition with { Page = currentCondition.Page + 1 };
```

小さな値型では、`readonly record struct` とファクトリを組み合わせると、値ベース等価性と validation を両立しやすくなります。

```csharp
public readonly record struct Percentage
{
    public Percentage(decimal value)
    {
        if (value is < 0 or > 100)
        {
            throw new ArgumentOutOfRangeException(nameof(value));
        }

        Value = value;
    }

    public decimal Value { get; }
}
```

## 避けたい書き方

大きな可変データを `struct` にすると、コピーコストや意図しない値コピーでバグを生みやすくなります。また、Entity を `record` にしてしまうと、値が同じなら同じものと見なされるため、ドメイン上の同一性とずれることがあります。

```csharp
// 避けたい例: Entity の同一性が値ベースの等価性に引っ張られる
public record Order(Guid Id, string Status);
```

可変 `struct` は、プロパティの変更が元の値に反映されるのかコピーに対する変更なのかが見えにくくなります。特にプロパティやコレクション越しに扱うと、意図と違う場所を更新しているように見えることがあります。

```csharp
// 避けたい例: 値コピーされる型なのに変更可能
public struct Price
{
    public decimal Amount { get; set; }
    public string Currency { get; set; }
}
```

`with` で簡単にコピーできることが、必ずしも良い設計とは限りません。状態遷移で監査ログ、イベント発行、validation が必要なら、専用メソッドを通す方が意図が残ります。

```csharp
// 避けたい例: 状態遷移の意味がただのコピー更新に見える
var canceled = order with { Status = OrderStatus.Canceled };
```

## 改善例

ID で識別するものは `class` にし、同一性の比較を明示します。値として扱う型は不変に寄せると、等価性と変更タイミングが読みやすくなります。

```csharp
public sealed class Order
{
    public required Guid Id { get; init; }
    public OrderStatus Status { get; private set; }

    public void Cancel()
    {
        Status = OrderStatus.Canceled;
    }
}

public readonly record struct Price(decimal Amount, string Currency);
```

状態変更に意味がある場合は、型のメソッドで表します。これにより、テストも「どの操作でどの状態に遷移するか」を確認しやすくなります。

```csharp
public void Cancel(IClock clock)
{
    if (Status is OrderStatus.Shipped)
    {
        throw new InvalidOperationException("出荷済みの注文はキャンセルできません。");
    }

    Status = OrderStatus.Canceled;
    CanceledAt = clock.Now;
}
```

## レビュー観点

- その型は ID で識別するのか、値の組み合わせで識別するのか
- `struct` が大きすぎないか、可変になっていないか
- `record` の自動生成される等価性が業務ルールと合っているか
- DTO、Value Object、Entity の責務が混ざっていないか
- `with` や deconstruction によって、重要な状態遷移が軽く見えすぎていないか
- `record` の public property がそのまま外部入力の形に引っ張られていないか
- `record struct` の `default` 値が不正状態として紛れ込まないか

## テスト観点

- Value Object の等価性が、業務上同じと見なす値の組み合わせに一致しているか
- Entity は同じ値を持つ別インスタンス、同じ ID を持つインスタンスをどう扱うか明確か
- `with` で作ったコピーが validation や状態遷移ルールを迂回していないか
- `struct` / `record struct` の `default`、境界値、大量コピー時の扱いに問題がないか

---

[Part README に戻る](README.md)


