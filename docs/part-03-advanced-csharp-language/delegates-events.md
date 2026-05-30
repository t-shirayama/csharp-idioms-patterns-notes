# delegates、events

## 概要

delegate は、メソッドを値として扱うための仕組みです。`Func<T>`、`Action<T>`、`Predicate<T>` のような組み込み delegate は、LINQ、コールバック、差し替え可能な処理で広く使われます。

event は、delegate を使った通知の公開方法です。外部から直接発火されないようにし、通知の所有権を発火側に残します。実務では、UI、ドメインイベント、進捗通知、長時間処理の状態変化などで登場します。

## 判断基準

- 小さな処理差し替えなら `Func` / `Action` を使う。
- その処理に業務上の名前や複数メソッドが必要なら interface を検討する。
- 状態変化を複数の購読者へ通知したいなら event を使う。
- 発火順、例外、非同期、購読解除が重要なら、event より明示的な mediator や message bus を検討する。
- handler が長寿命オブジェクトに購読されるなら、解除漏れによるメモリリークを設計で防ぐ。

## 使いどころ

delegate は、validation 条件、変換処理、検索条件、再試行時の実処理、テスト時の差し替えに向いています。interface を作るほどではない単一メソッドの依存を渡すと、テストしやすい小さな設計になります。

event は、発火側が「何か起きた」ことだけを通知し、購読側を知らない状態にしたいときに使います。ただし、イベントは処理の流れが追いにくくなりやすいため、業務上重要な同期処理を隠しすぎないことが大切です。

## 良い例

```csharp
public sealed class RetryPolicy
{
    public async Task<T> ExecuteAsync<T>(
        Func<CancellationToken, Task<T>> operation,
        CancellationToken cancellationToken)
    {
        for (var attempt = 1; attempt <= 3; attempt++)
        {
            try
            {
                return await operation(cancellationToken);
            }
            catch when (attempt < 3)
            {
                await Task.Delay(TimeSpan.FromMilliseconds(100), cancellationToken);
            }
        }

        throw new InvalidOperationException("Unreachable retry state.");
    }
}
```

`operation` を delegate として受け取ることで、再試行の制御と実処理を分離できます。テストでは失敗回数や cancellation を差し替えやすくなります。

event は、発火をクラス内に閉じる形で公開します。

```csharp
public sealed class ImportJob
{
    public event EventHandler<ProgressChangedEventArgs>? ProgressChanged;

    public void ReportProgress(int percentage)
    {
        ProgressChanged?.Invoke(
            this,
            new ProgressChangedEventArgs(percentage, null));
    }
}
```

購読側は通知を受け取れますが、外部から `ProgressChanged` を直接発火できません。

## 避けたい書き方

```csharp
public Action<string>? OnCompleted { get; set; }
```

これは外部から上書きでき、意図しないタイミングで呼び出すこともできます。通知として公開したいなら event の方が所有権が明確です。

```csharp
public event EventHandler? Saved;

public void Save()
{
    Saved?.Invoke(this, EventArgs.Empty);
    database.SaveChanges();
}
```

保存前に `Saved` を発火しているため、イベント名と実際の状態がずれています。購読側が保存済みを期待するとバグになります。

## 改善例

通知を「処理完了後」に寄せ、イベント名と状態を一致させます。

```csharp
public event EventHandler? Saved;

public void Save()
{
    database.SaveChanges();
    Saved?.Invoke(this, EventArgs.Empty);
}
```

購読解除が必要な場合は、ライフサイクルを明示します。

```csharp
public sealed class ProgressSubscription : IDisposable
{
    private readonly ImportJob job;

    public ProgressSubscription(ImportJob job)
    {
        this.job = job;
        this.job.ProgressChanged += OnProgressChanged;
    }

    public void Dispose()
    {
        job.ProgressChanged -= OnProgressChanged;
    }

    private static void OnProgressChanged(object? sender, ProgressChangedEventArgs e)
    {
        Console.WriteLine(e.ProgressPercentage);
    }
}
```

長寿命の publisher に短寿命の subscriber がぶら下がる場合、解除責任をコードに表すことが重要です。

## レビュー観点

- `Func` / `Action` で処理の意味が十分伝わるか。名前付き delegate や interface が必要でないか。
- event は外部から直接発火できない形で公開されているか。
- イベント名と発火タイミングの状態が一致しているか。
- 複数 handler のうち1つが例外を投げたときの扱いは許容できるか。
- 購読解除が必要なライフサイクルで、解除漏れを防げるか。
- テストで callback / event の呼び出し回数、引数、順序、例外時の挙動を確認できるか。
