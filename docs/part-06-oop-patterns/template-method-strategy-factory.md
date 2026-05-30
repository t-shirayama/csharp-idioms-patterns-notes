# Template Method、Strategy、Factory

## 概要

設計パターンは、よくある変更点に名前を付けて整理するための語彙です。Template Method は処理の骨格を固定し、Strategy はアルゴリズムを差し替え、Factory は生成手順や具象型選択を呼び出し側から隠します。

## 使いどころ

- Template Method: 処理順序は同じで、一部の手順だけ派生型ごとに違う。
- Strategy: 計算方法、料金計算、検証ルールなどを実行時や設定で切り替えたい。
- Factory: 生成に条件分岐、設定、外部リソース、複雑な初期化が絡む。
- パターン名よりも「どの変更点を閉じ込めたいか」を先に決める。

```csharp
public interface IDiscountStrategy
{
    decimal Apply(decimal price);
}

public sealed class StudentDiscount : IDiscountStrategy
{
    public decimal Apply(decimal price) => price * 0.9m;
}

public sealed class PriceCalculator
{
    private readonly IDiscountStrategy _discount;

    public PriceCalculator(IDiscountStrategy discount)
    {
        _discount = discount;
    }

    public decimal Calculate(decimal price) => _discount.Apply(price);
}
```

## 判断基準

Template Method は処理順序を基底クラスで固定するため、手順の入れ替えが少ない処理に向きます。派生型ごとに順序まで変わるなら、Strategy やパイプラインとして分けるほうが見通しがよくなります。

Strategy は、呼び出し側が「どのアルゴリズムを使うか」を外から選べる状態にしたいときに使います。単なる `if` が 2 つあるだけなら過剰ですが、料金、税、配送、認可のようにルール単位でテストしたい場合は効果があります。

Factory は、生成時の分岐や依存解決を呼び出し側から隠したいときに使います。生成後の業務処理まで Factory に入れると、責務が膨らみます。

```csharp
public sealed class DiscountStrategyFactory
{
    public IDiscountStrategy Create(CustomerRank rank) =>
        rank switch
        {
            CustomerRank.Student => new StudentDiscount(),
            CustomerRank.Premium => new PremiumDiscount(),
            _ => new NoDiscount()
        };
}
```

## 避けたい書き方

- 小さな `if` で十分な箇所にパターン名を当てはめ、クラス数だけを増やす。
- Factory が巨大な `switch` になり、結局すべての具象型を知っている。
- Strategy の粒度が細かすぎて、処理の流れがファイルをまたいで追いにくい。
- Template Method の hook が多くなり、派生型がどの組み合わせを実装すべきか分からない。

```csharp
// 避けたい例: Factory が業務処理まで持ち始めている。
public sealed class PaymentFactory
{
    public async Task ChargeAsync(Order order, CancellationToken cancellationToken)
    {
        var gateway = order.IsForeignCurrency ? new GlobalGateway() : new DomesticGateway();
        await gateway.ChargeAsync(order.Total, cancellationToken);
    }
}
```

## よくある失敗

- パターンを入れたあとも呼び出し側に分岐が残り、抽象化の効果が出ていない。
- Strategy の実装が DI コンテナ登録に散らばり、どの条件で選ばれるか追いにくい。
- Factory が `IServiceProvider` を直接受け取り、Service Locator と同じ見え方になる。
- Template Method の基底クラスを変更したら、すべての派生型の前提が崩れる。

## レビュー観点

- 変更される可能性が高い部分だけを分離しているか。
- パターン導入前よりテスト対象が明確になっているか。
- クラス名がパターン名ではなく業務上の意味を表しているか。
- Factory の責務が「生成」に収まり、業務処理まで持っていないか。
- 条件追加時に、既存コードの修正が局所的で済むか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: Strategy / Factory / Template Method の選択表

> 対応元: P1 / Strategy / Factory / Template Method の選択表、単純な `switch` で十分な場合、DI 登録と選択ロジックの置き場所を追加する。

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


## TODO対応: Decorator

> 対応元: P2 / Decorator、Adapter、Specification、Repository など、実務で頻出する関連パターンを「詳説する / しない」の整理として追加する。

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
