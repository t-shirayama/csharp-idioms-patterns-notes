# null 処理

## 概要

null 処理は、単なるクラッシュ回避ではなく、値が存在しないことを設計として表すかどうかの判断です。`??`、`?.`、`??=`、`ArgumentNullException.ThrowIfNull` は、前提と代替値を読み手に伝えるために使います。

Nullable reference types が有効なコードでは、型注釈はレビューの契約になります。`string?` は「null が仕様としてあり得る」、`string` は「入口で守るべき前提」として扱い、警告を消すためだけの記号にしないことが大切です。

## 判断基準

null が「あり得ない前提違反」なら入口で例外にします。null が「検索結果なし」や「任意入力なし」を表すなら、nullable annotation で契約として示します。null が「未読み込み」「権限なし」「外部システム障害」など複数の意味を持ち始めたら、専用型や Result 型を検討します。

`?.` は便利ですが、必須データの欠落を静かに通す用途には向きません。代替値を入れる場合も、その代替値が業務上自然か、単にエラーを隠していないかを確認します。

null と空文字、空コレクションは同じではありません。入力欄では null と空白を同じ「未入力」として正規化することがありますが、検索結果では「該当なし」と「検索していない」を分けたい場合があります。意味が違うなら、同じ `null` に押し込めない方が安全です。

`!` null 許容抑制演算子は、フレームワークやテストフィクスチャの都合で前提を表す最後の手段です。使う場合は、直前のガードやライフサイクル上の保証が読み取れる場所に限定します。

## 使いどころ

- 必須引数は入口で `ArgumentNullException.ThrowIfNull` により落とす。
- 表示名など、代替値が自然な場面では `??` を使う。
- コレクションは null ではなく空で返すと、呼び出し側の分岐を減らしやすい。
- 任意入力は、境界で trim や空白扱いを決めてから内部へ渡す。
- 外部 API や DB の欠損値は、変換境界で null の意味を明示する。

```csharp
public string GetDisplayName(User user)
{
    ArgumentNullException.ThrowIfNull(user);

    return user.Profile?.DisplayName ?? user.Email;
}
```

検索結果なしを表すなら、戻り値の nullable annotation で契約にします。

```csharp
public Customer? FindCustomer(CustomerId id)
{
    return customers.FirstOrDefault(customer => customer.Id == id);
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

コレクションを返す API は、原則として空コレクションを返すと呼び出し側が扱いやすくなります。ただし「未取得」と「取得済みだが0件」を分ける必要がある場合は、状態を別に持たせます。

```csharp
public IReadOnlyList<Order> GetRecentOrders(CustomerId customerId)
{
    return ordersByCustomer.TryGetValue(customerId, out var orders)
        ? orders
        : [];
}
```

## 避けたい書き方

- どこでも `?.` を使い、必須データの欠落を静かに通してしまう。
- null を返すメソッドと空コレクションを返すメソッドが混在している。
- `!` null 許容抑制演算子で警告だけを消し、実行時の安全性を確認していない。
- null、空文字、空白を場当たり的に同じ扱いにする。
- 「未取得」「権限なし」「障害」をすべて null で表す。

```csharp
var name = user!.Profile!.DisplayName!;
```

```csharp
// 避けたい例: 本来必須の Customer 欠落が「不明」に化ける
var customerName = order.Customer?.Name ?? "不明";
```

`??` の既定値が自然に見えても、データ欠損を隠すなら危険です。

```csharp
// 避けたい例: 税率設定の欠落が 0% として処理される
var taxRate = settings.TaxRate ?? 0m;
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

既定値が業務上自然ではない場合は、設定欠落として失敗させます。任意値として許す場合だけ、名前や型でその意味を表します。

```csharp
public decimal GetTaxRate(TaxSettings settings)
{
    return settings.TaxRate
        ?? throw new InvalidOperationException("税率設定が未設定です。");
}
```

## レビュー観点

- null が「異常」「未設定」「検索結果なし」のどれを表すのか明確か。
- public API では nullable annotation が戻り値と引数の契約を表しているか。
- `?.` の結果が null になった後の振る舞いをテストしているか。
- Optional 的な表現が必要なほど状態が複雑なら、専用型や Result 型を検討したか。
- 空文字、空コレクション、null の扱いが API ごとにばらついていないか。
- null 合体演算子で入れた既定値が、障害やデータ欠落を隠していないか。
- `!` を使う場合、非 null になる保証が近くに説明されているか。
- 外部入力を内部モデルへ変換する境界で、null の意味を正規化しているか。

## テスト観点

- 必須引数の null は例外になり、任意引数の null は仕様どおり変換されることを分けて確認する。
- `??` の既定値が入るケースを、正常な代替値としてテスト名に明示する。
- `?.` で null が伝播する場合、呼び出し側でどう扱われるかまで確認する。
- 空文字、空白、null を同じ扱いにするなら、3ケースをまとめて押さえる。
- 検索結果なし、未取得、権限なしなど、似た状態を null で混同していないかをケースで確認する。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: 重複を整理し

> 対応元: P1 / 重複を整理し、Part 1 は nullable contract / 型設計、Part 2 は `??`、`?.`、境界での正規化、Result/Option 代替に寄せる。

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
