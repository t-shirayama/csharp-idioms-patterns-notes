# delegates と events

## 概要

delegate は、メソッドを値として扱うための仕組みです。`Func<T>`、`Action<T>`、`Predicate<T>` のような組み込み delegate は、LINQ、コールバック、差し替え可能な処理で広く使われます。

event は、delegate を使った通知の公開方法です。外部から直接発火されないようにし、通知の所有権を発火側に残します。実務では、UI、ドメインイベント、進捗通知、長時間処理の状態変化などで登場します。

## 判断基準

- 小さな処理差し替えなら `Func` / `Action` を使う。
- その処理に業務上の名前や複数メソッドが必要なら interface を検討する。
- 状態変化を複数の購読者へ通知したいなら event を使う。
- 発火順、例外、非同期、購読解除が重要なら、event より明示的な mediator や message bus を検討する。
- handler が長寿命オブジェクトに購読されるなら、解除漏れによるメモリリークを設計で防ぐ。
- delegate の戻り値や例外が制御フローの一部なら、名前付き delegate や policy 型で意図を明示する。
- `async void` event handler は例外を捕まえにくい。非同期処理を待つ必要があるなら `Func<T, Task>` や明示的な dispatcher を検討する。

## 使いどころ

delegate は、validation 条件、変換処理、検索条件、再試行時の実処理、テスト時の差し替えに向いています。interface を作るほどではない単一メソッドの依存を渡すと、テストしやすい小さな設計になります。

event は、発火側が「何か起きた」ことだけを通知し、購読側を知らない状態にしたいときに使います。ただし、イベントは処理の流れが追いにくくなりやすいため、業務上重要な同期処理を隠しすぎないことが大切です。

delegate は依存を「関数1つ」に縮める設計です。差し替えが簡単になる一方で、名前が `Func<Order, bool>` だけだと業務上の意味が薄くなります。レビューでは、処理の意味が変数名だけで足りるか、型名として残すべきかを確認します。

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

戻り値が業務判断として重要なら、組み込み delegate より名前を付ける方が読みやすくなります。

```csharp
public delegate bool CanApproveOrder(Order order, User user);

public sealed class ApprovalService(CanApproveOrder canApprove)
{
    public bool CanApprove(Order order, User user)
    {
        return canApprove(order, user);
    }
}
```

単なる `Func<Order, User, bool>` より、何を判断している delegate なのかがレビューで伝わります。

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

非同期処理を event に隠すのも注意が必要です。

```csharp
public event EventHandler? Completed;

public void Complete()
{
    Completed?.Invoke(this, EventArgs.Empty);
}
```

購読側が `async void` handler で外部I/Oを始めると、発火側は完了や失敗を待てません。通知に失敗してよいのか、処理全体の成否に含めるべきなのかを分けて設計します。

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

非同期の処理差し替えは、event ではなく delegate を引数として明示する方がテストしやすい場合があります。

```csharp
public sealed class ImportRunner(
    Func<ImportResult, CancellationToken, Task> notifyCompleted)
{
    public async Task RunAsync(CancellationToken cancellationToken)
    {
        var result = await ImportAsync(cancellationToken);
        await notifyCompleted(result, cancellationToken);
    }
}
```

発火側が通知完了を待つ設計なら、例外と cancellation を通常の async フローとして扱えます。

## 使いすぎのサイン

- `Func` / `Action` の引数が多く、呼び出し側でラムダの意味を読み解く必要がある。
- event handler が業務処理の本体になり、どこで状態が変わるのか追えない。
- 購読解除の責務がコメントにしか書かれていない。
- handler の例外を握りつぶすか伝播するかが発火側で決まっていない。
- 非同期処理を `async void` event handler に押し込み、失敗時の観測先がない。

## レビュー観点

- `Func` / `Action` で処理の意味が十分伝わるか。名前付き delegate や interface が必要でないか。
- event は外部から直接発火できない形で公開されているか。
- イベント名と発火タイミングの状態が一致しているか。
- 複数 handler のうち1つが例外を投げたときの扱いは許容できるか。
- 購読解除が必要なライフサイクルで、解除漏れを防げるか。
- テストで callback / event の呼び出し回数、引数、順序、例外時の挙動を確認できるか。

## テスト観点

- callback が呼ばれない条件、1回だけ呼ばれる条件、複数回呼ばれる条件を確認する。
- delegate が例外を投げたとき、呼び出し側が再試行、伝播、握りつぶしのどれを選ぶか。
- event は未購読、単一購読、複数購読、解除後の発火を確認する。
- handler の1つが例外を投げた場合、後続 handler を呼ぶ設計かどうかを確認する。
- 非同期 callback は cancellation と例外伝播を確認し、`async void` に依存したテストにしない。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: 非同期 callback の Func<..., ValueTask>

> 対応元: P1 / 非同期 callback の `Func<..., ValueTask>`、`CancellationToken` を含む delegate 設計、event より mediator/channel を選ぶ場面を追加する。

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
