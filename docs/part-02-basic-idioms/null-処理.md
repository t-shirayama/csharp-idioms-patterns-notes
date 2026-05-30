# null 処理

## 概要

null 処理は、単なるクラッシュ回避ではなく、値が存在しないことを設計として表すかどうかの判断です。`??`、`?.`、`??=`、`ArgumentNullException.ThrowIfNull` は、前提と代替値を読み手に伝えるために使います。

## 判断基準

null が「あり得ない前提違反」なら入口で例外にします。null が「検索結果なし」や「任意入力なし」を表すなら、nullable annotation で契約として示します。null が「未読み込み」「権限なし」「外部システム障害」など複数の意味を持ち始めたら、専用型や Result 型を検討します。

`?.` は便利ですが、必須データの欠落を静かに通す用途には向きません。代替値を入れる場合も、その代替値が業務上自然か、単にエラーを隠していないかを確認します。

## 使いどころ

- 必須引数は入口で `ArgumentNullException.ThrowIfNull` により落とす。
- 表示名など、代替値が自然な場面では `??` を使う。
- コレクションは null ではなく空で返すと、呼び出し側の分岐を減らしやすい。

```csharp
public string GetDisplayName(User user)
{
    ArgumentNullException.ThrowIfNull(user);

    return user.Profile?.DisplayName ?? user.Email;
}
```

## 良い例

必須値は早めにガードし、任意値は変換時に意味を整えます。null と空文字を同じ扱いにするかは、入力仕様として決めておきます。

```csharp
public string NormalizeDisplayName(string? displayName, string email)
{
    ArgumentException.ThrowIfNullOrWhiteSpace(email);

    return string.IsNullOrWhiteSpace(displayName)
        ? email
        : displayName.Trim();
}
```

遅延初期化では `??=` が読みやすい場面があります。ただし、共有状態やマルチスレッドで扱う値では、単純な `??=` だけで十分かを確認します。

```csharp
private List<string>? _warnings;

public void AddWarning(string message)
{
    (_warnings ??= []).Add(message);
}
```

## 避けたい書き方

- どこでも `?.` を使い、必須データの欠落を静かに通してしまう。
- null を返すメソッドと空コレクションを返すメソッドが混在している。
- `!` null 許容抑制演算子で警告だけを消し、実行時の安全性を確認していない。

```csharp
var name = user!.Profile!.DisplayName!;
```

```csharp
// 避けたい例: 本来必須の Customer 欠落が「不明」に化ける
var customerName = order.Customer?.Name ?? "不明";
```

## 改善例

必須関連が欠けているなら、境界で検出して失敗にします。検索結果なしの場合は、戻り値の型で「ないかもしれない」ことを示します。

```csharp
public string GetCustomerName(Order order)
{
    ArgumentNullException.ThrowIfNull(order.Customer);
    return order.Customer.Name;
}

public Customer? FindCustomer(CustomerId id)
{
    return customers.FirstOrDefault(customer => customer.Id == id);
}
```

## レビュー観点

- null が「異常」「未設定」「検索結果なし」のどれを表すのか明確か。
- public API では nullable annotation が戻り値と引数の契約を表しているか。
- `?.` の結果が null になった後の振る舞いをテストしているか。
- Optional 的な表現が必要なほど状態が複雑なら、専用型や Result 型を検討したか。
- 空文字、空コレクション、null の扱いが API ごとにばらついていないか。
- null 合体演算子で入れた既定値が、障害やデータ欠落を隠していないか。

---

[Part README に戻る](README.md)


