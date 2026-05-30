# 型、値型、参照型、nullable reference types

## 概要

C# では値型 (value type) と参照型 (reference type) の違いが、null、コピー、等価性、パフォーマンスに影響します。C# 8 以降では nullable reference types を有効にして、「null になりうるか」を型で表すのが基本です。

- `int` や `DateTime` は値型で、通常は null にならない
- `int?` のような nullable value type は「値がない」を表せる
- `string` は null 非許容、`string?` は null 許容として読む
- null 警告はコンパイラからの設計レビューとして扱う

## 判断基準

`?` を付けるかどうかは「実際に値がない状態を許すか」で決めます。DB、JSON、画面入力、外部 API の境界では null が入り得るため、受け口で `?` を明示します。ドメイン内部やサービス内部では、早めに検証して null 非許容の型に変換すると、後続の条件分岐が減ります。

値型はコピーされ、参照型は参照が渡されます。ただし、実務では「スタックかヒープか」よりも、変更可能性、等価性、null の扱い、コレクションに入れたときの振る舞いを意識した方がレビューしやすくなります。

## 使いどころ

外部入力、DB、JSON、古い API など、null が混ざる境界では `?` を明示します。ドメイン内部では、可能な限り null 非許容に寄せると、呼び出し側の分岐が減ってテストしやすくなります。

```csharp
public sealed class UserProfile
{
    public required string DisplayName { get; init; }
    public string? Bio { get; init; }
}

public static string GetLabel(UserProfile profile)
{
    return profile.Bio is { Length: > 0 }
        ? $"{profile.DisplayName} ({profile.Bio})"
        : profile.DisplayName;
}
```

## 良い例

境界では nullable を受け止め、内部に入る前にガードします。`Try` パターンを使う場合は、成功時に値が null でないことを属性で伝えると、呼び出し側の警告を減らせます。

```csharp
public static bool TryCreate(
    string? value,
    [NotNullWhen(true)] out EmailAddress? email)
{
    if (string.IsNullOrWhiteSpace(value) || !value.Contains('@'))
    {
        email = null;
        return false;
    }

    email = new EmailAddress(value);
    return true;
}
```

## 避けたい書き方

null 警告を消すためだけに null-forgiving operator (`!`) を使うのは避けます。`!` は「ここでは null ではないと設計上保証できる」場合に限るべきです。

```csharp
// 避けたい例: 実際の null 可能性を隠している
var name = request.UserName!.Trim();
```

入力検証が必要なら、早い段階でガードしてから非 null として扱います。

```csharp
ArgumentException.ThrowIfNullOrWhiteSpace(request.UserName);
var name = request.UserName.Trim();
```

## 改善例

外部モデルの nullable を内部モデルへ変換する関数を置くと、null チェックが散らばりにくくなります。変換時点で失敗理由を返せる形にすると、入力エラーのテストも書きやすくなります。

```csharp
public static UserProfile CreateProfile(UpdateProfileRequest request)
{
    ArgumentException.ThrowIfNullOrWhiteSpace(request.DisplayName);

    return new UserProfile
    {
        DisplayName = request.DisplayName.Trim(),
        Bio = string.IsNullOrWhiteSpace(request.Bio) ? null : request.Bio.Trim()
    };
}
```

## レビュー観点

- nullable reference types が有効になっているか
- `string?` と `string` の違いが設計意図として説明できるか
- `!` で警告を隠していないか
- 外部境界で null を受け止め、内部モデルでは null を減らせているか
- `default` 値で作られた値型が業務上有効か、不正状態にならないか
- nullable 警告を抑制する属性を使う場合、実装と属性の契約が一致しているか

---

[Part README に戻る](README.md)


