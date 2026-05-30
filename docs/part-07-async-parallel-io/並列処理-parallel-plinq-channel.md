# 並列処理、Parallel、PLINQ、Channel

## 概要

並行 (concurrency) は複数の処理を同時期に進める設計で、並列 (parallelism) は複数コアで同時に実行することです。I/O-bound な処理は `async` / `await` が中心で、CPU-bound な処理は `Parallel` や PLINQ を検討します。生産者と消費者を分けたい場合は `Channel` が有力です。

```csharp
var options = new ParallelOptions
{
    MaxDegreeOfParallelism = Environment.ProcessorCount,
    CancellationToken = cancellationToken
};

await Parallel.ForEachAsync(files, options, async (file, token) =>
{
    var image = await imageLoader.LoadAsync(file, token);
    await thumbnailWriter.WriteAsync(image, token);
});
```

## 使いどころ

- CPU-bound な計算や変換を複数コアに分散する。
- 大量データをパイプライン処理し、入力と出力の速度差を吸収する。
- `Channel` でバックプレッシャーを持たせ、無制限なタスク生成を避ける。
- I/O を並行実行する場合も、外部サービスや DB の許容量に合わせて上限を置く。

```csharp
var throttler = new SemaphoreSlim(8);

var tasks = userIds.Select(async id =>
{
    await throttler.WaitAsync(cancellationToken);
    try
    {
        return await userClient.GetSummaryAsync(id, cancellationToken);
    }
    finally
    {
        throttler.Release();
    }
});

var summaries = await Task.WhenAll(tasks);
```

## 判断基準

CPU-bound な処理は、コア数、処理時間、データサイズを見て並列化します。小さな処理を無理に並列化すると、スケジューリングや同期のコストで遅くなることがあります。

I/O-bound な処理は、`Parallel.ForEach` でブロックするより、`Task.WhenAll`、`Parallel.ForEachAsync`、`Channel` などで非同期に待つほうが自然です。ただし、無制限に同時実行すると外部サービス、DB 接続プール、ファイルハンドルを詰まらせます。

`Channel` は、読み取りと書き込みを分け、キュー長に上限を持たせたいときに有効です。単にリストを並列変換したいだけなら、より単純な API を選びます。

## 避けたい書き方

- I/O 待ちを速くしたいだけなのに、`Parallel.ForEach` で同期的にブロックする。
- 共有 `List<T>` や `Dictionary<TKey, TValue>` に複数スレッドから書き込む。
- `Task.Run` を大量に投げ、同時実行数を制御しない。
- ASP.NET Core のリクエスト処理で、CPU-bound な長時間処理を安易に `Task.Run` へ逃がす。

```csharp
// 避けたい例: thread-safe ではないコレクションへ並列に書き込んでいる。
var results = new List<Result>();
Parallel.ForEach(items, item =>
{
    results.Add(Convert(item));
});
```

```csharp
// 避けたい例: 外部 API への同時実行数が入力件数に比例する。
var tasks = ids.Select(id => client.GetAsync($"/users/{id}", cancellationToken));
await Task.WhenAll(tasks);
```

## よくある失敗

- 並列化で順序が変わることを考慮せず、結果の比較や保存順に依存する。
- 例外発生時に残りのタスクを止める方針がなく、不要な外部 I/O が続く。
- `lock` で共有状態を守った結果、ほとんど直列実行になっている。
- PLINQ で副作用のある処理を実行し、順序やスレッド安全性の問題を起こす。

## レビュー観点

- 対象処理が CPU-bound か I/O-bound かを分けて考えているか。
- 同時実行数に上限があるか。
- 共有状態へのアクセスが `lock`、不変データ、thread-safe collection などで保護されているか。
- 並列化による順序の変化が、業務結果やテストに影響しないか。
- 例外、キャンセル、部分成功時に、投入済み作業をどう扱うか決まっているか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: PLINQ の専用節を追加し

> 対応元: P0 / PLINQ の専用節を追加し、`AsParallel`、`WithDegreeOfParallelism`、`AsOrdered`、副作用禁止、例外集約を扱う。

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
