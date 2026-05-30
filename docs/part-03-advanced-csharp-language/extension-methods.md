# extension methods

## 概要

extension method は、既存の型にメソッドを追加したように呼び出せる静的メソッドです。フレームワーク型、外部ライブラリの型、DTO、`IEnumerable<T>` などに補助的な操作を足すときに便利です。

ただし、実際には静的メソッドであり、対象型の責務が増えるわけではありません。便利さの裏で、依存が見えにくくなる、業務ロジックが散らばる、namespace によって使えるAPIが変わる、という問題も起こります。

## 判断基準

- 対象型の自然な補助操作として読めるなら extension method を検討する。
- 対象型を変更できない場合や、横断的な表示・変換・判定を足したい場合に向く。
- 状態変更、外部I/O、重要な業務ルールを隠すなら避ける。
- 呼び出し側が namespace を見なくても意図を推測できる名前にする。
- public にする前に、通常の static helper や domain service の方が自然でないか確認する。

## 使いどころ

extension method は、文字列やコレクションの小さな判定、DTO から表示用テキストへの変換、テストデータ作成の補助、`IServiceCollection` への登録拡張などでよく使います。

特に `IServiceCollection` や `IEndpointRouteBuilder` のように、フレームワークが拡張ポイントとして想定している型では、extension method が自然なAPIになります。

## 良い例

```csharp
public static class StringExtensions
{
    public static bool IsNullOrWhiteSpace(this string? value)
    {
        return string.IsNullOrWhiteSpace(value);
    }
}
```

null を受け取る可能性を `string?` で表し、実装も安全です。ただし、この例は標準APIとほぼ同じなので、実プロジェクトでは本当に必要か確認します。

フレームワークの登録APIでは、責務が分かりやすい単位にまとめると効果的です。

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddOrderApplication(
        this IServiceCollection services)
    {
        services.AddScoped<OrderUseCase>();
        services.AddScoped<OrderValidator>();
        return services;
    }
}
```

呼び出し側は `services.AddOrderApplication()` と読め、登録の詳細を1箇所に閉じ込められます。

## 避けたい書き方

```csharp
public static class OrderExtensions
{
    public static void Approve(this Order order, DbContext dbContext)
    {
        order.Status = OrderStatus.Approved;
        dbContext.SaveChanges();
    }
}
```

状態変更とDB保存が extension method に隠れています。呼び出し側からは軽いメソッドに見えますが、実際には外部I/Oを含むため、レビューやテストで見落としやすくなります。

```csharp
public static bool IsValid(this string value)
{
    return value.Length > 0;
}
```

名前が一般的すぎて、何に対して valid なのか分かりません。validation ルールは文脈を持つ名前にします。

## 改善例

業務状態の変更は、Entity のメソッドや application service に置きます。

```csharp
public sealed class Order
{
    public OrderStatus Status { get; private set; }

    public void Approve()
    {
        if (Status != OrderStatus.Submitted)
        {
            throw new InvalidOperationException("Submitted order only.");
        }

        Status = OrderStatus.Approved;
    }
}
```

DB保存は use case 側で扱います。

```csharp
public sealed class ApproveOrderUseCase
{
    public async Task ExecuteAsync(Guid orderId, CancellationToken cancellationToken)
    {
        var order = await repository.FindAsync(orderId, cancellationToken);
        order.Approve();
        await repository.SaveAsync(order, cancellationToken);
    }
}
```

extension method は、責務が軽い変換や登録に留めます。

```csharp
public static string ToDisplayName(this OrderStatus status)
{
    return status switch
    {
        OrderStatus.Submitted => "申請中",
        OrderStatus.Approved => "承認済み",
        OrderStatus.Rejected => "却下",
        _ => status.ToString()
    };
}
```

## レビュー観点

- 対象型に自然に属する補助操作として読めるか。
- 外部I/O、DB保存、HTTP通信、ログ出力などの副作用を隠していないか。
- namespace import により、意図しない場所で利用可能にならないか。
- メソッド名は文脈を持ち、既存APIと衝突しにくいか。
- null 許容性と引数 validation は明確か。
- テストで境界値、null、空コレクション、対象 enum の未定義値を確認できるか。
