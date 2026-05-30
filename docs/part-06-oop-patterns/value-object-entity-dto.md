# Value Object、Entity、DTO

## 概要

Value Object は値の等価性で扱う型、Entity は識別子によって同一性を持つ型、DTO は層やプロセスの境界を越えてデータを運ぶ型です。`record` は Value Object や DTO と相性がよい一方、Entity に使う場合は等価性の意味に注意が必要です。

## 使いどころ

- Value Object: メールアドレス、金額、期間など、値とルールをまとめたい。
- Entity: ユーザー、注文、請求書など、状態が変わっても同じものとして追跡したい。
- DTO: Web API、DB、メッセージングなど、境界の入出力を安定させたい。
- 型名だけで単位、制約、ライフサイクルを読み取れるようにしたい。

```csharp
public readonly record struct Money(decimal Amount, string Currency)
{
    public Money
    {
        if (Amount < 0)
        {
            throw new ArgumentOutOfRangeException(nameof(Amount));
        }

        if (string.IsNullOrWhiteSpace(Currency))
        {
            throw new ArgumentException("通貨コードは必須です。", nameof(Currency));
        }
    }
}
```

## 判断基準

Value Object は、値が同じなら同じものとして扱ってよい型です。生成時に不正値を弾き、できれば immutable にします。`record` や `readonly record struct` は候補になりますが、内部に可変コレクションを持つと等価性の見え方が崩れます。

Entity は、属性が変わっても識別子が同じなら同じものとして扱います。`record` の自動等価性は全プロパティを比較するため、Entity では意図とずれやすい点に注意します。

DTO は境界用の形です。入力 DTO は外部から来るため検証前提、出力 DTO は互換性前提で設計します。ドメインモデルをそのまま JSON に出すと、内部変更が外部 API 変更になります。

```csharp
public sealed class Customer
{
    public CustomerId Id { get; }
    public EmailAddress Email { get; private set; }

    public void ChangeEmail(EmailAddress email)
    {
        Email = email;
    }
}
```

## 避けたい書き方

- DTO に業務ロジックを詰め込み、API 変更とドメイン変更が連動する。
- Entity を `record` にして、全プロパティの一致を同一性として扱ってしまう。
- `string` や `decimal` をそのまま渡し続け、単位や妥当性が呼び出し側に散らばる。
- EF Core の都合で作った形を、そのままドメインモデルや API 契約として広げる。

```csharp
// 避けたい例: 通貨や丸めのルールが呼び出し側に散る。
public void Charge(decimal amount)
{
    if (amount < 0)
    {
        throw new ArgumentOutOfRangeException(nameof(amount));
    }
}
```

## よくある失敗

- DTO と Entity のプロパティ名が同じなので mapping を省略し、境界の意味まで失う。
- Value Object に `default` 値で不正状態を作れる余地がある。
- `record` の `with` 式で、検証を通らない組み合わせが作れる設計にしてしまう。
- ID を `Guid` のまま各所に渡し、顧客 ID と注文 ID の取り違えをコンパイル時に防げない。

## レビュー観点

- その型は同一性で見るのか、値の等価性で見るのか。
- DTO とドメインモデルを直接共有して、外部仕様が内部設計を縛っていないか。
- Value Object の生成時に不正な値を防げているか。
- mapping が冗長すぎる場合でも、境界を消してよい理由があるか。
- `record`、`class`、`struct` の選択が、等価性と可変性の意味に合っているか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: DTO

> 対応元: P0 / DTO、Entity、Value Object がドメイン層・アプリケーション層・インフラ層のどこに属するか、入力 DTO から Value Object を生成して Entity に渡す流れ、出力 DTO への mapping 例を追加する。

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
