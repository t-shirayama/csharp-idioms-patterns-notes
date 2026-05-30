# BenchmarkDotNet の入口

## 概要

BenchmarkDotNet は .NET の micro benchmark を安定して実行するためのライブラリです。JIT、warmup、Release build、複数回実行などを扱ってくれるため、手元の `Stopwatch` より信頼しやすい結果を得られます。

ただし、micro benchmark は局所的な差を見る道具です。実際の遅さが DB、ネットワーク、ロック、GC、アルゴリズム、UI 描画のどこにあるかを確認してから使います。

```csharp
using BenchmarkDotNet.Attributes;

public class StringBuildBenchmarks
{
    private readonly string[] values = ["A", "B", "C"];

    [Benchmark]
    public string Join() => string.Join(",", values);

    [Benchmark]
    public string Interpolation() => $"{values[0]},{values[1]},{values[2]}";
}
```

## 判断基準

- まず実アプリのメトリクス、ログ、プロファイラで遅い箇所を絞る。
- BenchmarkDotNet は、局所的な実装差を比較するときに使う。
- 入力サイズ、データ分布、失敗ケースを本番に近づける。
- 平均時間だけでなく、allocation、ばらつき、外れ値、コード複雑化を見る。
- benchmark のために現実と違う前提を置く場合、その制約を明記する。

## 使いどころ

- 文字列処理、collection 操作、パーサー、変換処理など、局所的な実装差を比較する。
- 「速いはず」というレビュー上の主張を、再現可能な測定で確認する。
- allocation、平均時間、ばらつきを見て、最適化の効果と副作用を判断する。

## 避けたい書き方

- Debug build、デバッガ接続、1 回だけの `Stopwatch` 計測で結論を出す。
- 実アプリの遅さを測らず、関係の薄い micro benchmark だけを改善する。
- 入力データが現実と違いすぎる benchmark を根拠にする。
- 結果の差が小さいのに、可読性を大きく落とす実装へ置き換える。
- benchmark 対象の戻り値を使わず、JIT の最適化で処理が消える形にする。

```csharp
// 避けたい例: 現実の入力分布がなく、結果も使われない。
[Benchmark]
public void Measure()
{
    string.Join(",", new[] { "A", "B", "C" });
}
```

```csharp
[MemoryDiagnoser]
public class CsvBenchmarks
{
    private readonly Order[] orders = TestData.CreateOrders(1_000);

    [Benchmark]
    public string JoinCodes()
    {
        return string.Join(",", orders.Select(order => order.Code));
    }
}
```

## レビュー観点

- 先にプロファイルやログでボトルネック候補を絞っているか。
- benchmark が Release build で実行され、JIT や warmup の影響を考慮しているか。
- 入力サイズ、データ分布、失敗ケースが実運用に近いか。
- 平均時間だけでなく、allocation、外れ値、コードの複雑化も見て判断しているか。
- benchmark の結果を PR に貼る場合、環境、.NET version、実行条件が分かるか。
- micro benchmark の改善が、実アプリのメトリクス改善につながる前提を説明できるか。

---

[Part README に戻る](README.md)
