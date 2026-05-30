# ProblemDetails とエラー応答

## 概要

Web API のエラー応答は、利用者と運用者の両方に向けた契約です。ASP.NET Core では `ProblemDetails` を使うことで、HTTP status、エラーの種類、説明、発生箇所、trace id を一貫した形で返せます。

重要なのは、例外メッセージをそのまま返すことではありません。利用者が次に何をすればよいか、運用者がログと照合できるか、クライアントが安定して分岐できるかを分けて考えます。

```csharp
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = context =>
    {
        context.ProblemDetails.Extensions["traceId"] =
            context.HttpContext.TraceIdentifier;
    };
});

app.UseExceptionHandler();
app.UseStatusCodePages();
```

## 判断基準

利用者が修正できる入力エラーは 400、認証が必要なら 401、権限不足は 403、存在しないリソースは 404、競合は 409、レート制限は 429、想定外の失敗は 500 として扱います。

`detail` には利用者が理解できる範囲の説明を入れ、内部例外、SQL、stack trace、接続先、secret は出しません。クライアントが機械的に分岐する必要がある場合は、安定した `type` やアプリケーション固有の error code を使います。

## 使いどころ

- API 全体のエラー JSON 形式をそろえる。
- validation error と domain error と system error を分ける。
- ログの trace id とレスポンスを結びつける。
- クライアントが status code と error code で安定して分岐できるようにする。

```csharp
public static class ProblemResults
{
    public static IResult Conflict(string code, string detail) =>
        Results.Problem(
            statusCode: StatusCodes.Status409Conflict,
            title: "Conflict",
            detail: detail,
            type: $"https://example.com/problems/{code}");
}
```

## 良い例

業務上の失敗を、HTTP と `ProblemDetails` の公開契約へ明示的に変換します。

```csharp
app.MapPost("/users", async (
        CreateUserRequest request,
        CreateUserUseCase useCase,
        CancellationToken cancellationToken) =>
    {
        var result = await useCase.CreateAsync(request, cancellationToken);

        return result switch
        {
            CreateUserResult.Success success =>
                Results.Created($"/users/{success.UserId}", success),

            CreateUserResult.EmailAlreadyUsed =>
                Results.Problem(
                    statusCode: StatusCodes.Status409Conflict,
                    title: "Email already used",
                    type: "https://example.com/problems/email-already-used"),

            _ => Results.Problem(statusCode: StatusCodes.Status500InternalServerError)
        };
    });
```

## 避けたい書き方

例外の中身をそのまま JSON にすると、内部情報を漏らし、クライアント契約も不安定になります。

```csharp
// 避けたい例: 内部例外をそのまま公開している。
catch (Exception ex)
{
    return Results.Json(new
    {
        error = ex.Message,
        stackTrace = ex.StackTrace
    }, statusCode: 500);
}
```

また、すべての失敗を 200 で返すと、HTTP の意味が失われ、監視やクライアント実装が複雑になります。

## 改善例

公開してよい情報と、ログに残す情報を分けます。

```csharp
public sealed class DuplicateEmailException(string email) : Exception
{
    public string Email { get; } = email;
}

public sealed class DuplicateEmailExceptionHandler(
    IProblemDetailsService problemDetailsService,
    ILogger<DuplicateEmailExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        if (exception is not DuplicateEmailException duplicate)
        {
            return false;
        }

        logger.LogInformation("Duplicate email rejected: {Email}", duplicate.Email);
        httpContext.Response.StatusCode = StatusCodes.Status409Conflict;

        return await problemDetailsService.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = httpContext,
            ProblemDetails =
            {
                Title = "Email already used",
                Status = StatusCodes.Status409Conflict,
                Type = "https://example.com/problems/email-already-used"
            }
        });
    }
}
```

## レビュー観点

- エラー応答の形式が API 全体で統一されているか。
- 利用者向けの説明と内部調査用のログを混同していないか。
- status code が失敗の種類に合っているか。
- validation、認証、認可、404、409、429、500 の扱いが一貫しているか。
- trace id や correlation id がレスポンスとログをつないでいるか。
- クライアントが依存する error code や `type` が安定した契約になっているか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: validation error

> 対応元: P0 / validation error、model binding error、認証/認可失敗、例外ハンドラの統合テスト例を追加する。

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
