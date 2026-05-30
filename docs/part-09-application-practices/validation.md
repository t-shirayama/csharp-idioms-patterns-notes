# 入力検証 (Validation)

## 概要

validation は、入力の形を確認する処理と、業務ルールを守る処理に分けて考えます。リクエスト DTO の必須項目や文字数は入力検証、在庫数や状態遷移の妥当性は domain validation として扱うと、責務が混ざりにくくなります。

DataAnnotations は単純な検証に向いています。複数項目の関係や条件付き検証が増える場合は FluentValidation などを使うと、ルールを Controller から分離しやすくなります。

```csharp
public sealed record CreateUserRequest(
    string DisplayName,
    string Email);

public sealed class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.DisplayName).NotEmpty().MaximumLength(40);
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
    }
}
```

## 使いどころ

- API の request DTO で、必須、桁数、形式、範囲を早めに弾きたい。
- 画面や API から同じ入力ルールを使いたい。
- 業務ルールの失敗を例外ではなく validation error として利用者へ返したい。

## 入力検証と業務検証

validation は層ごとに役割を分けると見通しがよくなります。

- DTO validation: 必須、文字数、形式、範囲など、入力の形を確認する。
- Application validation: 認証ユーザーの権限、対象リソースの存在、操作可能な状態を確認する。
- Domain validation: Entity や Value Object が守るべき不変条件を確認する。

```csharp
public sealed record Money
{
    public decimal Amount { get; }

    public Money(decimal amount)
    {
        if (amount < 0)
        {
            throw new ArgumentOutOfRangeException(nameof(amount));
        }

        Amount = amount;
    }
}
```

Value Object の不変条件は、DTO の属性だけに任せません。API 以外の入口、バッチ、テスト、将来の UI から作られても守られる必要があるためです。

## エラーの返し方

利用者が修正できる入力不備は、例外ではなく validation error として返す方が自然です。Web API では `ProblemDetails` や `ValidationProblemDetails` を使うと、HTTP の契約として扱いやすくなります。

```csharp
if (!result.IsValid)
{
    return Results.ValidationProblem(result.ToDictionary());
}
```

ただし、存在確認や権限確認を validation に混ぜすぎると、攻撃者に「その ID が存在するか」を教えてしまう場合があります。404、403、400 のどれで返すかは、利用者体験だけでなく情報開示の観点でも決めます。

## よい例

入力の形は DTO validator で、業務上の状態遷移は domain model で扱います。

```csharp
public sealed class Order
{
    public OrderStatus Status { get; private set; }

    public bool CanCancel() => Status is OrderStatus.Draft or OrderStatus.Submitted;

    public void Cancel()
    {
        if (!CanCancel())
        {
            throw new InvalidOperationException("Order cannot be cancelled.");
        }

        Status = OrderStatus.Cancelled;
    }
}
```

この例外はユーザー入力の文字数不備ではなく、ドメインの前提違反です。アプリケーション層で先に `CanCancel()` を見て、利用者には業務エラーとして返す設計もあります。

## 避けたい書き方

- Controller、Service、Domain Model の各所に同じ入力検証を重複して書く。
- ユーザーが修正できる入力ミスを例外として扱い、500 系エラーにしてしまう。
- domain validation を DTO の属性だけで表そうとして、業務ルールが外から見えなくなる。
- validator から DB や外部 API を何度も呼び、入力検証を重い処理にする。
- エラーメッセージに内部項目名や SQL 制約名をそのまま出す。

```csharp
// 避けたい例: 入力不備なのにシステム例外として扱われやすい。
if (request.Quantity <= 0)
{
    throw new InvalidOperationException("Quantity must be positive.");
}
```

```csharp
// 避けたい例: DTO の都合がドメインの不変条件を代替している。
public sealed class OrderEntity
{
    [Range(0, int.MaxValue)]
    public int Quantity { get; set; }
}
```

## レビュー観点

- 入力検証と業務ルールの検証が適切な層に分かれているか。
- 利用者が修正できるエラーを validation error として返しているか。
- 同じルールが複数箇所に重複していないか。
- エラーメッセージが利用者向けか、ログ・開発者向けかを混同していないか。
- validation が外部 I/O に強く依存し、遅い・不安定な入口になっていないか。
- 境界値、null、空文字、空配列、重複などがテストされているか。
- API のエラー応答がクライアントにとって機械的に扱える形になっているか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: model binding 失敗

> 対応元: P1 / model binding 失敗、invalid JSON、nullable、DB 一意制約との競合、async validation の扱いを追加する。

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
