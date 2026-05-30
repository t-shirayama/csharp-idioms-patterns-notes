# 時刻、乱数、外部 I/O の分離

## 概要

時刻、乱数、ファイル、HTTP、DB は、テストを不安定にしやすい代表的な依存です。C# 8 以降の現代的なコードでは、`.NET 8` 以降なら `TimeProvider` を使う選択肢があります。プロジェクトの対象フレームワークによっては、自前の clock interface を小さく用意するのも現実的です。

## 使いどころ

- 期限、リトライ、キャッシュ、有効期間など、現在時刻で結果が変わる処理。
- 乱数や ID 採番を含む処理を、テストでは固定値にしたい場面。
- ファイル、HTTP、DB を直接触る処理を adapter に寄せ、業務ルールと分けたい場面。

```csharp
public sealed class SubscriptionPolicy
{
    private readonly TimeProvider timeProvider;

    public SubscriptionPolicy(TimeProvider timeProvider)
    {
        this.timeProvider = timeProvider;
    }

    public bool CanRenew(DateTimeOffset expiresAt)
    {
        var now = timeProvider.GetUtcNow();
        return expiresAt <= now.AddDays(30);
    }
}
```

## 時刻の扱い

時刻は UTC、ローカル時刻、業務上のタイムゾーンを分けます。保存や比較は `DateTimeOffset` または UTC を基本にし、表示や締め日の判定でタイムゾーンを明示します。

```csharp
public sealed class TrialPolicy
{
    private readonly TimeProvider timeProvider;

    public TrialPolicy(TimeProvider timeProvider)
    {
        this.timeProvider = timeProvider;
    }

    public bool IsActive(DateTimeOffset startedAt)
    {
        return timeProvider.GetUtcNow() < startedAt.AddDays(14);
    }
}
```

`.NET 8` 以降では `FakeTimeProvider` を使うと、時間を進めるテストが書きやすくなります。対象フレームワークが古い場合は、`IClock` のような小さな interface で十分です。

## 乱数と ID

乱数や ID は「値そのものが重要」なのか「一意であることが重要」なのかを分けます。テストで固定したいなら generator を注入し、業務ロジックから直接 `Guid.NewGuid()` を呼ばないようにします。

```csharp
public interface IIdGenerator
{
    Guid NewId();
}

public sealed class FixedIdGenerator : IIdGenerator
{
    private readonly Guid id;

    public FixedIdGenerator(Guid id)
    {
        this.id = id;
    }

    public Guid NewId() => id;
}
```

セキュリティ用途の乱数は、テスト容易性だけで `Random` に置き換えてはいけません。本番実装では `RandomNumberGenerator` など用途に合う API を選び、テストではその境界を差し替えます。

## 外部 I/O の境界

外部 I/O は adapter に寄せ、業務処理には「必要な操作」だけを見せます。`HttpClient` やファイルパスを深い層まで渡すと、失敗ケース、リトライ、タイムアウトの扱いが散らばります。

```csharp
public interface IInvoiceStorage
{
    Task SaveAsync(InvoiceFile file, CancellationToken cancellationToken);
}
```

adapter は薄く保ち、業務判断を入れすぎないようにします。外部サービスのレスポンスを domain に変換する場所は必要ですが、支払い可否や注文状態の判断まで adapter に入れるとテストの境界が曖昧になります。

## 避けたい書き方

- テスト内で `Thread.Sleep` を使い、時間経過を待って検証する。
- `new Random()` をメソッド内で毎回作り、結果が固定できない。
- 業務ロジックから `File.ReadAllText` や `HttpClient` を直接呼び、失敗ケースを作りにくくする。
- ローカル時刻と UTC を混ぜて比較し、環境や夏時間で結果が変わる。
- cancellation や timeout を adapter で握りつぶし、呼び出し元に伝えない。

```csharp
// 避けたい例: 実行環境のタイムゾーンに依存する。
return DateTime.Now.Date == dueDate.Date;
```

```csharp
// 避けたい例: テストが遅く不安定になりやすい。
Thread.Sleep(TimeSpan.FromSeconds(1));
Assert.True(cache.IsExpired(key));
```

## レビュー観点

- 時刻は UTC、ローカル時刻、タイムゾーンのどれで扱うか明確か。
- 乱数や ID 生成が、テストで固定できる境界に置かれているか。
- HTTP や DB の adapter が薄く、業務判断が混ざっていないか。
- deterministic な単体テストと、本物の I/O を使う統合テストの役割が分かれているか。
- `CancellationToken`、timeout、retry の責務が境界で一貫しているか。
- セキュリティ用途の乱数と、テスト用の固定値生成を混同していないか。

---

[Part README に戻る](README.md)


