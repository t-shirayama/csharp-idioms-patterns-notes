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

<!-- TODO対応追記 -->

## TODO対応: MemoryDiagnoser

> 対応元: P1 / `MemoryDiagnoser`、`Params`、`GlobalSetup`、`Consumer`、baseline、結果表の読み方、環境情報の貼り方を追加する。

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
