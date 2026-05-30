# attributes

## 概要

attribute は、型、メソッド、プロパティ、引数などにメタデータを付ける仕組みです。C# では `[Obsolete]`、`[Serializable]`、nullable 関連属性、ASP.NET Core の routing / validation 属性、テストフレームワークの `[Fact]` など、多くの場所で使われます。

attribute 自体は「印」です。実際の動作は、reflection、source generator、analyzer、フレームワークの実行時処理など、attribute を読む側によって決まります。そのため、attribute を付けるだけで何が保証されるのかを誤解しないことが大切です。

## 判断基準

- 宣言的なメタデータとして表すと読みやすいなら attribute を使う。
- 実行時処理が必要な場合は、attribute を読む仕組みと失敗時の挙動を明確にする。
- 業務ロジックの本体を attribute に隠しすぎるなら、通常のコードや service を優先する。
- reflection を使う場合は、性能、AOT、trimming、テスト容易性を確認する。
- compile-time に検出したいなら、source generator や analyzer との組み合わせを検討する。

## 使いどころ

attribute は、validation、serialization、routing、authorization、DI登録のマーカー、ログ除外、生成コードの対象指定などに向いています。コードの近くに宣言を置けるため、設定ファイルや別テーブルよりも読みやすい場合があります。

一方で、attribute は処理の流れを直接表しません。実行時にどのタイミングで読まれるのか、複数 attribute が競合した場合にどうなるのかを確認できる設計にします。

## 良い例

```csharp
public sealed class CreateUserRequest
{
    [Required]
    [StringLength(100)]
    public string Name { get; init; } = "";

    [EmailAddress]
    public string Email { get; init; } = "";
}
```

入力DTOの validation では、attribute によって制約が近くに見えます。ただし、業務上の一意性や複数項目にまたがるルールは、別の validator に分ける方が自然です。

独自 attribute は、読む側とセットで設計します。

```csharp
[AttributeUsage(AttributeTargets.Class, AllowMultiple = false)]
public sealed class AuditTargetAttribute : Attribute
{
    public AuditTargetAttribute(string name)
    {
        Name = name;
    }

    public string Name { get; }
}
```

`AttributeUsage` を指定すると、どこに何回付けられるかが明確になります。

## 避けたい書き方

```csharp
[AttributeUsage(AttributeTargets.Method)]
public sealed class RetryAttribute : Attribute
{
    public int Count { get; set; } = 3;
}
```

attribute を定義しただけでは retry は動きません。AOP、filter、source generator など、誰がこの attribute を読んで処理するのかが必要です。

```csharp
[BusinessRule("premium-user-can-skip-approval")]
public void Approve()
{
    Status = OrderStatus.Approved;
}
```

業務ルールの中身が文字列メタデータに逃げると、コンパイル時に守れず、テストも追いにくくなります。

## 改善例

attribute は対象指定に留め、実処理は明示的なコードに置きます。

```csharp
[AuditTarget("orders")]
public sealed class OrderService
{
    public Task ApproveAsync(Guid orderId, CancellationToken cancellationToken)
    {
        return approveOrderUseCase.ExecuteAsync(orderId, cancellationToken);
    }
}
```

起動時に attribute の読み取りを検証できると安全です。

```csharp
public static IReadOnlyList<Type> FindAuditTargets(Assembly assembly)
{
    return assembly.GetTypes()
        .Where(type => type.GetCustomAttribute<AuditTargetAttribute>() is not null)
        .ToList();
}
```

reflection を使う処理は小さく閉じ込め、テストで attribute あり・なし・重複などを確認します。

## レビュー観点

- attribute はメタデータとして自然か。通常のコードの方が読みやすくないか。
- attribute を読む仕組みは明確か。起動時やテストで検証できるか。
- `AttributeUsage` で対象、継承、複数指定のルールが表されているか。
- reflection 使用時の性能、AOT、trimming への影響を確認しているか。
- attribute の引数に、環境依存値や変更頻度の高い値を無理に入れていないか。
- 業務ルールが文字列やメタデータに隠れて、型安全性を失っていないか。
