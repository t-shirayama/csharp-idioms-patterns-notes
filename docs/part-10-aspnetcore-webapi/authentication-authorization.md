# Authentication / Authorization

## 概要

Authentication は「利用者が誰か」を確認する処理で、Authorization は「その利用者が何をしてよいか」を判断する処理です。ASP.NET Core ではこの2つを別の middleware と policy として扱えるため、混ぜずに設計することが重要です。

実務では、ログイン済みかどうかだけでは不十分です。管理者か、対象テナントに属しているか、対象データの所有者か、必要な scope を持っているかまで確認します。

```csharp
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer();

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("Orders.Read", policy =>
        policy.RequireClaim("scope", "orders.read"));
});

app.UseAuthentication();
app.UseAuthorization();

app.MapGet("/orders/{id}", GetOrderAsync)
   .RequireAuthorization("Orders.Read");
```

## 判断基準

認証方式は API の利用者で選びます。ブラウザー中心の同一サイトなら cookie、外部クライアントや SPA なら JWT bearer、サービス間通信なら mTLS や client credentials を検討します。

認可は role だけで表現できるとは限りません。role は粗い分類に向き、claim や scope は外部 IdP との契約に向き、policy はアプリケーション内の判断名として使いやすいです。所有者チェックやテナント境界のようにデータを見ないと判断できないものは、use case 側でも確認します。

## 使いどころ

- endpoint 単位で必要な権限を宣言する。
- claim や scope を policy にまとめ、文字列を散らばらせない。
- テナント ID や所有者 ID を業務処理の境界で確認する。
- 401 と 403 を分け、認証失敗と認可失敗を混同しない。

```csharp
public static class Policies
{
    public const string OrdersWrite = "Orders.Write";
}

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy(Policies.OrdersWrite, policy =>
        policy.RequireAuthenticatedUser()
              .RequireClaim("scope", "orders.write"));
});
```

## 良い例

権限名を定数や拡張メソッドに寄せ、endpoint では意図が読める名前を使います。さらに、データ単位の認可は use case で確認します。

```csharp
app.MapPost("/orders/{id}/cancel", async (
        Guid id,
        ClaimsPrincipal user,
        CancelOrderUseCase useCase,
        CancellationToken cancellationToken) =>
    {
        var userId = UserId.FromClaims(user);
        var result = await useCase.CancelAsync(id, userId, cancellationToken);

        return result.ToHttpResult();
    })
   .RequireAuthorization(Policies.OrdersWrite);
```

## 避けたい書き方

認証済みであることだけを確認し、対象データへの権限確認を忘れる書き方は危険です。

```csharp
// 避けたい例: ログイン済みなら他人の注文も操作できてしまう。
app.MapDelete("/orders/{id}", async (Guid id, AppDbContext db) =>
{
    var order = await db.Orders.FindAsync(id);
    db.Orders.Remove(order!);
    await db.SaveChangesAsync();
    return Results.NoContent();
}).RequireAuthorization();
```

`User.IsInRole("Admin")` のような文字列判断を各所に散らすと、権限変更やテストが難しくなります。

## 改善例

endpoint では policy を宣言し、所有者やテナント境界は use case に閉じ込めます。

```csharp
public sealed class CancelOrderUseCase(AppDbContext db)
{
    public async Task<CancelOrderResult> CancelAsync(
        Guid orderId,
        UserId userId,
        CancellationToken cancellationToken)
    {
        var order = await db.Orders
            .SingleOrDefaultAsync(x => x.Id == orderId, cancellationToken);

        if (order is null)
        {
            return CancelOrderResult.NotFound();
        }

        if (order.UserId != userId)
        {
            return CancelOrderResult.Forbidden();
        }

        order.Cancel();
        await db.SaveChangesAsync(cancellationToken);
        return CancelOrderResult.Success();
    }
}
```

## レビュー観点

- authentication と authorization の責務が分かれているか。
- 匿名アクセスを許す endpoint が明示されているか。
- policy 名、claim 名、scope 名が散らばらず管理されているか。
- テナント境界、所有者確認、管理者権限がデータ単位で確認されているか。
- 401、403、404 の使い分けが情報漏えいと利用者体験の両方に合っているか。
- 認可ロジックがテスト可能な場所にあり、フレームワーク依存に埋もれていないか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: JWT bearer の検証設定

> 対応元: P0 / JWT bearer の検証設定、issuer/audience、clock skew、scope/role claim mapping、fallback policy、`AllowAnonymous` の扱いを追加する。

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


## TODO対応: cookie 認証時の CSRF

> 対応元: P0 / cookie 認証時の CSRF、SameSite、SPA/BFF 構成、API key の保管・ローテーションを追加する。

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
