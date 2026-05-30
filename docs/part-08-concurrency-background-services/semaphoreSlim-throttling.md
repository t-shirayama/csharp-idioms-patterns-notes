# SemaphoreSlim による同時実行数制御

## 概要

`SemaphoreSlim` は、同時に実行できる処理数を制限するための同期プリミティブです。`lock` が「同時に1つだけ入れる」排他を表すのに対して、`SemaphoreSlim` は「同時にN件まで」を表せます。

外部 API、DB、ファイル変換、重い計算など、入力件数ぶん一気に開始すると危険な処理では、上限をコード上に置くことが重要です。

## 判断基準

- 同時実行数を 1 より大きい値にしたいなら `SemaphoreSlim` を検討する。
- 待機が非同期になるなら `WaitAsync` を使う。
- `WaitAsync` に成功したら、必ず `finally` で `Release` する。
- 上限値は気分で決めず、DB 接続数、外部 API 制限、CPU、メモリを根拠にする。

## 使いどころ

大量のユーザーにメールを送る、複数ファイルを変換する、外部 API から詳細情報を取得する、DB 更新を一定数ずつ流す、といった「並行にはしたいが無制限にはしたくない」場面に向きます。

一方で、処理順序を厳密に守りたい、キュー長を管理したい、producer と consumer を分けたい場合は `Channel` の方が設計を表しやすいことがあります。

## 良い例

```csharp
public async Task<IReadOnlyList<CustomerProfile>> LoadProfilesAsync(
    IReadOnlyList<CustomerId> customerIds,
    CancellationToken cancellationToken)
{
    using var throttler = new SemaphoreSlim(initialCount: 8);

    var tasks = customerIds.Select(async id =>
    {
        await throttler.WaitAsync(cancellationToken);
        try
        {
            return await _client.GetProfileAsync(id, cancellationToken);
        }
        finally
        {
            throttler.Release();
        }
    });

    return await Task.WhenAll(tasks);
}
```

この例では、入力件数が多くても外部 API 呼び出しは最大 8 件までです。

## 避けたい書き方

```csharp
var tasks = customerIds.Select(id => _client.GetProfileAsync(id));
var profiles = await Task.WhenAll(tasks);
```

入力件数が 10 件なら問題にならなくても、1万件なら一気に1万リクエストを開始します。外部 API、socket、DB、メモリのどこかが先に詰まります。

## 改善例

上限値を設定から渡し、レビューや運用で調整できるようにします。

```csharp
public sealed class ProfileImportOptions
{
    public int MaxConcurrency { get; init; } = 8;
}

public async Task ImportAsync(
    IReadOnlyList<CustomerId> ids,
    ProfileImportOptions options,
    CancellationToken cancellationToken)
{
    using var throttler = new SemaphoreSlim(options.MaxConcurrency);

    await Task.WhenAll(ids.Select(async id =>
    {
        await throttler.WaitAsync(cancellationToken);
        try
        {
            var profile = await _client.GetProfileAsync(id, cancellationToken);
            await _repository.SaveAsync(profile, cancellationToken);
        }
        finally
        {
            throttler.Release();
        }
    }));
}
```

同時実行数は、処理の特性や外部制約によって変わります。コードに固定値を埋め込むより、設定として説明できる場所に置く方が運用しやすくなります。

## レビュー観点

- 無制限の `Task.WhenAll` になっていないか。
- `WaitAsync` と `Release` が必ず対応しているか。
- `CancellationToken` が `WaitAsync` と実処理の両方へ渡っているか。
- 上限値に根拠があるか。外部 API や DB の制限と矛盾していないか。
- 例外時に `Release` 漏れで処理全体が止まらないか。
- 順序保証やキュー管理が必要なら、`SemaphoreSlim` だけで十分か。

<!-- TODO対応追記 -->

## TODO対応: SemaphoreSlim(initialCount, maxCount)

> 対応元: P1 / `SemaphoreSlim(initialCount, maxCount)`、over-release 検出、巨大入力で全 task を先に作る問題と `Channel` / batching への切替基準を追加する。

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
