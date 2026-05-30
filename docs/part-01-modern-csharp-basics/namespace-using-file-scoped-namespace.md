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

---

[Part README に戻る](README.md)


