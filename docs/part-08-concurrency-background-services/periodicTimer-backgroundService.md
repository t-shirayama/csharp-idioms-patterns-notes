# PeriodicTimer / BackgroundService

## 概要

`BackgroundService` は、ASP.NET Core や Worker Service で常駐処理を実装するための基本クラスです。`PeriodicTimer` は、一定間隔で非同期処理を実行したいときに使いやすいタイマーです。

定期実行や常駐ワーカーでは、開始処理よりも停止、例外、キャンセル、前回処理との重なり、ログの設計が重要になります。

## 判断基準

- アプリケーションのライフサイクルに合わせて動く常駐処理なら `BackgroundService` を検討する。
- 一定間隔で非同期処理を実行するなら `PeriodicTimer` を使う。
- 前回処理が長引いたときに、次回処理を重ねるのか、待つのかを決める。
- 例外時にプロセスを止めるのか、ログを出して継続するのかを明示する。

## 使いどころ

定期的なキャッシュ更新、古いデータのクリーンアップ、外部キューのポーリング、期限切れジョブの処理、メトリクス収集などに向きます。

ただし、Web リクエスト中に時間のかかる処理を裏で流したいだけなら、単純な fire-and-forget ではなく、永続キューや `Channel`、外部ジョブ基盤を検討します。

## 良い例

```csharp
public sealed class CleanupWorker : BackgroundService
{
    private readonly ILogger<CleanupWorker> _logger;
    private readonly ExpiredSessionCleaner _cleaner;

    public CleanupWorker(
        ILogger<CleanupWorker> logger,
        ExpiredSessionCleaner cleaner)
    {
        _logger = logger;
        _cleaner = cleaner;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromMinutes(5));

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                await _cleaner.CleanAsync(stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                break;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Expired session cleanup failed.");
            }
        }
    }
}
```

この例では、前回の `CleanAsync` が終わるまで次の処理に入りません。処理が長引く場合、実行間隔より処理完了を優先する設計です。

## 避けたい書き方

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (true)
    {
        await Task.Delay(TimeSpan.FromMinutes(5));
        await _cleaner.CleanAsync(CancellationToken.None);
    }
}
```

停止要求を無視しており、アプリ終了時に待ち続ける可能性があります。`CancellationToken.None` を渡すと、下位処理も停止できません。

## 改善例

停止トークンを delay と実処理へ渡し、初回実行のタイミングも明示します。

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    await RunOnceAsync(stoppingToken);

    using var timer = new PeriodicTimer(TimeSpan.FromMinutes(5));

    while (await timer.WaitForNextTickAsync(stoppingToken))
    {
        await RunOnceAsync(stoppingToken);
    }
}

private async Task RunOnceAsync(CancellationToken stoppingToken)
{
    try
    {
        await _cleaner.CleanAsync(stoppingToken);
    }
    catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
    {
        throw;
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Cleanup failed.");
    }
}
```

初回にすぐ実行するのか、最初の tick まで待つのかは業務要件です。コード上で読み取れるようにします。

## レビュー観点

- `stoppingToken` が待機と実処理へ伝播しているか。
- 前回処理と次回処理が重なる可能性を理解しているか。
- 例外時に継続するのか、停止するのか決まっているか。
- 定期実行の間隔、初回実行、タイムアウト、リトライの方針が説明できるか。
- アプリ終了時に中断してよい処理か、完了待ちすべき処理か。
- ログから開始、完了、処理件数、処理時間、失敗理由を追えるか。

<!-- TODO対応追記 -->

## TODO対応: BackgroundService で scoped service を使う場合の …

> 対応元: P0 / `BackgroundService` で scoped service を使う場合の `IServiceScopeFactory` 例を追加する。

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


## TODO対応: fake clock

> 対応元: P2 / fake clock、短い interval、停止 token、例外時継続の検証例を追加する。

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
