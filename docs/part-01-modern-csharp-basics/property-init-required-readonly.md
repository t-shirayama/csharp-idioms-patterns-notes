# property、init、required、readonly

## 概要

プロパティとフィールドの設計は、オブジェクトの不変性と初期化の安全性に直結します。C# 9 以降の `init`、C# 11 以降の `required` を使うと、オブジェクト初期化子の読みやすさを保ちながら、未設定を減らせます。

- 自動実装プロパティは単純な状態に使う
- get-only property はコンストラクタで確定する値に向く
- init-only property は初期化後に変えたくない値に向く
- required member は必須値の設定漏れをコンパイル時に見つけやすくする
- readonly field はコンストラクタ後に差し替えない依存や値に使う

## 判断基準

状態を外から変更できる必要があるか、初期化時だけ決まればよいか、コンストラクタで前提を保証したいかで使い分けます。DTO や設定値のように単純な入れ物なら `required init` が読みやすく、ドメインオブジェクトのように不正状態を作りたくない型ではコンストラクタやファクトリで検証する方が安全です。

`required` は「オブジェクト初期化子で設定すること」を要求する機能であり、値の妥当性までは保証しません。`required string Name { get; init; }` は未設定を減らせますが、空文字、空白、形式不正は通ります。C# 11 以降の補助線として使い、validation や不変条件は別に置きます。

`init` は初期化後の再代入を防ぎますが、`init` setter の中で validation を書けます。単純な DTO では自動実装で十分ですが、型の内部に入る値を正規化したい場合は、backing field と組み合わせる選択もあります。

`readonly` は「再代入しない」ことを示すだけで、参照先オブジェクトの中身が不変になるわけではありません。`readonly List<T>` はリストの差し替えを防ぎますが、要素追加はできてしまいます。

## 使いどころ

設定値や DTO では `required` と `init` が有効です。サービスクラスの依存は `readonly` field に保持すると、途中で差し替わらないことが読み手に伝わります。

```csharp
public sealed class ReportOptions
{
    public required string OutputDirectory { get; init; }
    public int RetryCount { get; init; } = 3;
}

public sealed class ReportService
{
    private readonly IClock _clock;

    public ReportService(IClock clock)
    {
        _clock = clock;
    }
}
```

C# 12 の primary constructor を使う場合も、依存を保持する意味があるなら `readonly` field へ受けるか、コンストラクタ引数を直接使うかを意図して選びます。単に短くするためだけに導入すると、依存のライフサイクルやテスト差し替えが読みにくくなることがあります。

## 良い例

外部入力を受ける DTO と、内部で不変性を守りたい型を分けると責務が明確になります。`required` は設定漏れの検出に使い、業務上の妥当性は別の validation に寄せます。

```csharp
public sealed class CreateUserRequest
{
    public required string Email { get; init; }
    public string? DisplayName { get; init; }
}

public sealed class User
{
    public User(EmailAddress email)
    {
        Email = email;
    }

    public EmailAddress Email { get; }
}
```

`init` setter で正規化する場合は、例外を投げる条件を明確にし、外部入力モデルではなく内部モデル寄りの型に置くと読みやすくなります。

```csharp
public sealed class TeamName
{
    private string _value = "";

    public required string Value
    {
        get => _value;
        init
        {
            ArgumentException.ThrowIfNullOrWhiteSpace(value);
            _value = value.Trim();
        }
    }
}
```

## 避けたい書き方

どこからでも変更できる public setter を広げると、状態変更の原因を追いにくくなります。初期化後に変わらないはずの値は、`set` ではなく `init` やコンストラクタで表します。

```csharp
// 避けたい例: 必須値なのに未設定のまま作れて、後からも変更できる
public sealed class UserDto
{
    public string Name { get; set; } = "";
}
```

また、`required` は実行時 validation の代わりではありません。空文字や範囲不正は別途検証が必要です。

```csharp
// 避けたい例: readonly でもコレクションの中身は変更できる
private readonly List<string> _roles = [];

public IReadOnlyList<string> Roles => _roles;
```

`required` を付けた public mutable property も注意が必要です。初期化時に必須でも、その後どこからでも変更できるなら、不変条件の説明にはなりません。

```csharp
// 避けたい例: 必須値だが初期化後も自由に壊せる
public required string Email { get; set; }
```

## 改善例

可変コレクションは内部に閉じ込め、公開側は読み取り専用の型にします。初期化後に変更しない値は `init`、状態遷移として変更する値は private setter やメソッドで表します。

```csharp
private readonly List<string> _roles = [];

public IReadOnlyCollection<string> Roles => _roles.AsReadOnly();

public void AddRole(string role)
{
    ArgumentException.ThrowIfNullOrWhiteSpace(role);
    _roles.Add(role);
}
```

必須値と妥当性の両方が重要なら、コンストラクタやファクトリで受けると、作成時に必ず検証できます。`required` はシリアライズや設定 binding のように、オブジェクト初期化子との相性がよい場所で使うと効果が出やすくなります。

```csharp
public sealed class EmailAddress
{
    public EmailAddress(string value)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(value);
        Value = value.Contains('@') ? value : throw new ArgumentException("メールアドレス形式ではありません。");
    }

    public string Value { get; }
}
```

## レビュー観点

- public setter が本当に必要か
- 必須値が `required`、コンストラクタ、validation のどれで守られているか
- 初期化後に変わらない値が `init` や `readonly` で表現されているか
- `required` だけで空文字や不正値まで保証した気になっていないか
- コレクションを公開するとき、呼び出し側から変更できる形になっていないか
- private setter が単なる妥協ではなく、状態遷移メソッドとセットで設計されているか
- `required` と nullable reference types の組み合わせが、未設定と不正値を混同していないか

## テスト観点

- `required` では防げない空文字、範囲外、不正な組み合わせを validation で検出できるか
- `init` setter やコンストラクタで正規化した値が、期待どおり保存されるか
- public setter、private setter、状態遷移メソッドのどこから値を変えられるかがテストで追えるか
- 公開した読み取り専用コレクション越しに内部状態を変更できないか

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: C# 14 field backed properties

> 対応元: P0 / C# 14 `field` backed properties、`required` と constructor / serializer / configuration binding の相性、`SetsRequiredMembers` の注意点を追加する。 / `required` を validation と誤解しやすい点を具体例で補強する。

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
