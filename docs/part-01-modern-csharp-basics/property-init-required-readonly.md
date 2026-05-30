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

## レビュー観点

- public setter が本当に必要か
- 必須値が `required`、コンストラクタ、validation のどれで守られているか
- 初期化後に変わらない値が `init` や `readonly` で表現されているか
- `required` だけで空文字や不正値まで保証した気になっていないか
- コレクションを公開するとき、呼び出し側から変更できる形になっていないか
- private setter が単なる妥協ではなく、状態遷移メソッドとセットで設計されているか

---

[Part README に戻る](README.md)


