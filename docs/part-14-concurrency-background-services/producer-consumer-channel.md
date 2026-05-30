# producer / consumer と Channel

## 概要

`Channel<T>` は、producer が投入した item を consumer が非同期に処理するためのキューです。処理の入口と実行を分けられるため、Web リクエストから重い処理を切り離す、複数ワーカーで順次処理する、投入側に backpressure をかける、といった設計に使えます。

特に bounded channel は、キュー長に上限を置けます。処理が追いつかないときに無限にメモリを使うのではなく、投入側を待たせる、古い item を捨てる、書き込みを失敗させるなどの方針を選べます。

## 判断基準

- producer と consumer の責務を分けたいか。
- 入力速度と処理速度に差があり、キューで吸収したいか。
- キュー長、満杯時の動作、処理順序、完了通知を設計したいか。
- アプリ終了時に残った item を処理するのか、再実行対象にするのか決められるか。

## 使いどころ

メール送信、画像変換、取り込みジョブ、ログやイベントの後続処理、外部 API への送信など、呼び出し元と処理本体を分離したい場面に向きます。

ただし、プロセスが落ちたら失われて困るジョブは、インメモリの `Channel` だけでは不十分です。永続キュー、DB、メッセージブローカーなどを検討します。

## 良い例

```csharp
public sealed class WorkQueue
{
    private readonly Channel<WorkItem> _channel =
        Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(100)
        {
            FullMode = BoundedChannelFullMode.Wait
        });

    public ValueTask EnqueueAsync(WorkItem item, CancellationToken cancellationToken)
    {
        return _channel.Writer.WriteAsync(item, cancellationToken);
    }

    public IAsyncEnumerable<WorkItem> ReadAllAsync(CancellationToken cancellationToken)
    {
        return _channel.Reader.ReadAllAsync(cancellationToken);
    }
}
```

bounded channel にすると、処理が追いつかないとき producer が待ちます。これは意図的な backpressure です。

## 避けたい書き方

```csharp
private readonly Channel<WorkItem> _channel = Channel.CreateUnbounded<WorkItem>();

public void Enqueue(WorkItem item)
{
    _channel.Writer.TryWrite(item);
}
```

unbounded channel は、投入が処理を大きく上回るとメモリを使い続けます。また、`TryWrite` の戻り値を見ないと、失敗時に item を失ったことに気づけません。

## 改善例

consumer 側を `BackgroundService` として実装し、キャンセルと例外を扱います。

```csharp
public sealed class WorkQueueWorker : BackgroundService
{
    private readonly WorkQueue _queue;
    private readonly ILogger<WorkQueueWorker> _logger;

    public WorkQueueWorker(WorkQueue queue, ILogger<WorkQueueWorker> logger)
    {
        _queue = queue;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var item in _queue.ReadAllAsync(stoppingToken))
        {
            try
            {
                await ProcessAsync(item, stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                throw;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Work item {WorkItemId} failed.", item.Id);
            }
        }
    }
}
```

失敗した item をログだけで終えるのか、再試行するのか、dead-letter 相当の場所へ移すのかは業務要件です。ここを曖昧にすると、バックグラウンド処理は静かにデータを失います。

## レビュー観点

- channel が bounded か unbounded か、理由を説明できるか。
- 満杯時の `FullMode` が要件に合っているか。
- `WriteAsync` と `ReadAllAsync` に `CancellationToken` が渡っているか。
- producer 完了時に writer を `Complete` する必要があるか。
- consumer 例外時に item を失うのか、再試行するのか、失敗として記録するのか。
- インメモリ queue で失われて困らない処理か。永続化が必要ではないか。
- キュー長、処理時間、失敗数をログやメトリクスで追えるか。
