# namespace、using、file-scoped namespace

## 概要

名前空間は、コードの論理的な置き場所を表します。C# 10 以降では file-scoped namespace によってインデントを浅くでき、通常の業務コードでは読みやすさの面で有利です。

- namespace はプロジェクト構造と概ね対応させる
- `using` は必要なものだけ残す
- global using はプロジェクト全体で当然使うものに絞る
- file-scoped namespace は 1 ファイル 1 namespace の通常ケースに向く

## 判断基準

namespace は物理フォルダの写しではなく、依存方向と責務を読むための名前です。ただし、フォルダと大きくずれると探索性が落ちるため、原則は近づけます。例外として、生成コード、移行中の互換層、複数プロジェクトから共有する名前空間などは、意図をコメントやプロジェクト構成で補います。

`global using` は「ほぼ全ファイルで自然に使うもの」に限定します。便利だからという理由でアプリケーション層や Infrastructure 層の名前空間を広げると、参照している依存が見えにくくなります。

## 使いどころ

アプリケーションの層や機能に合わせて名前空間を切ると、責務の境界が見えやすくなります。たとえば `Orders.Application`、`Orders.Domain`、`Orders.Infrastructure` のように分けると、依存方向のレビューにも使えます。

```csharp
namespace Orders.Application;

public sealed class SubmitOrderUseCase
{
    public Task ExecuteAsync(SubmitOrderCommand command, CancellationToken cancellationToken)
    {
        // use case の調整処理を書く
        return Task.CompletedTask;
    }
}
```

## 良い例

層ごとの namespace を明確にしておくと、「Domain が Infrastructure を参照していないか」「Application が UI 固有の型に依存していないか」をコードレビューで見つけやすくなります。

```csharp
namespace Orders.Domain;

public sealed record OrderId(Guid Value);
```

```csharp
namespace Orders.Infrastructure.Persistence;

public sealed class OrderRepository
{
}
```

## 避けたい書き方

フォルダと名前空間が大きくずれていると、クラスを探しにくくなります。また、global using に業務固有の名前空間や頻繁に衝突する型を入れすぎると、依存が見えにくくなります。

```csharp
// 避けたい例: あらゆるファイルから Infrastructure が見えてしまう
global using Orders.Infrastructure.Database;
```

## 改善例

広く使う標準ライブラリやプロジェクト共通のテスト補助程度に留め、業務上の境界をまたぐ using は各ファイルで明示します。

```csharp
// GlobalUsings.cs
global using Microsoft.Extensions.Logging;
global using System.Diagnostics.CodeAnalysis;
```

```csharp
using Orders.Application.Ports;

namespace Orders.Infrastructure.Persistence;
```

## レビュー観点

- フォルダ構成と namespace が大きく乖離していないか
- global using が便利さより依存の見えにくさを生んでいないか
- 不要な using が残っていないか
- file-scoped namespace を使える単純なファイルで余計なネストが増えていないか
- 同名の型が増えて、using alias で補うより名前設計を見直すべき状態になっていないか
- テストプロジェクトの namespace が本体コードの構造と対応しているか

## テスト観点

- テストプロジェクトの namespace が本体コードと対応し、対象クラスを探しやすいか
- `InternalsVisibleTo` やテスト用 helper の namespace が、本体の依存方向を曖昧にしていないか
- global using に追加した依存が、テストから本体へ逆流していないか
- using alias を使うテストで、同名型の取り違えが起きないよう期待する型を明示しているか

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: SDK の implicit global usings

> 対応元: P2 / SDK の implicit global usings、C# 12 alias any type、file-local type の配置方針を追加する。

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
