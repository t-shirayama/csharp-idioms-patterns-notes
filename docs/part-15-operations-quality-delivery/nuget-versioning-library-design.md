# NuGet、versioning、library design

## 概要

NuGet パッケージを配布する場合、利用者はコードだけでなく、version、public API、依存 package、target framework、README、license、symbols、Source Link に依存します。ライブラリ設計では、使いやすい API を作るだけでなく、将来の互換性を壊しにくい形にすることが重要です。

SemVer は便利ですが、数字を上げればよいという話ではありません。public API の削除、シグネチャ変更、nullable 注釈変更、既定値変更、例外の種類変更も利用者にとっては breaking change になり得ます。

## 判断基準

- public API は一度公開すると簡単には消せない前提で設計する。
- patch は bug fix、minor は後方互換な追加、major は breaking change に使う。
- prerelease は利用者に安定性の期待値を伝えるために使う。
- 依存 package の version 範囲は、利用者側の解決と互換性を考えて決める。
- README、XML documentation、Source Link、symbols は利用者の導入と調査を助ける。

## 使いどころ

社内共通ライブラリ、SDK、クライアントライブラリ、汎用ユーティリティ、Source Generator、Analyzer、NuGet.org へ公開するパッケージで使います。アプリケーション内部の共通プロジェクトでも、複数チームが参照するなら同じ観点が必要です。

## 良い例

配布物として必要な情報を `.csproj` に明示します。

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <PackageId>Contoso.Ordering</PackageId>
    <Version>1.4.0</Version>
    <Authors>Contoso</Authors>
    <Description>Ordering primitives and validation helpers.</Description>
    <PackageReadmeFile>README.md</PackageReadmeFile>
    <PackageLicenseExpression>MIT</PackageLicenseExpression>
    <RepositoryUrl>https://github.com/contoso/ordering</RepositoryUrl>
    <PublishRepositoryUrl>true</PublishRepositoryUrl>
    <IncludeSymbols>true</IncludeSymbols>
    <SymbolPackageFormat>snupkg</SymbolPackageFormat>
  </PropertyGroup>
</Project>
```

## 避けたい書き方

利用者が継承や実装を前提にできてしまう API を不用意に公開すると、後から変更しづらくなります。

```csharp
public class OrderValidator
{
    public virtual bool IsValid(Order order) => true;
}
```

`virtual` は拡張点として強い契約になります。内部都合で変更する可能性があるなら、安易に公開しない方が安全です。

## 改善例

拡張点は interface と option で明示し、実装クラスは必要以上に継承させません。

```csharp
public interface IOrderValidator
{
    ValidationResult Validate(Order order);
}

public sealed class DefaultOrderValidator : IOrderValidator
{
    public ValidationResult Validate(Order order)
    {
        return order.Total >= 0
            ? ValidationResult.Success
            : ValidationResult.Failure("Total must be non-negative.");
    }
}
```

継承ではなく interface を契約にすると、利用者が依存する面積を小さくできます。

## レビュー観点

- version 更新が変更内容と SemVer に合っているか。
- public API の削除、引数追加、戻り値変更、nullable 注釈変更が breaking change として扱われているか。
- `public`、`protected`、`virtual`、`abstract` が本当に公開契約として必要か。
- target framework が利用者の環境に合い、不要に新しすぎないか。
- 依存 package の version 範囲が、利用者側の dependency resolution を壊しにくいか。
- README、release notes、XML documentation、Source Link、symbols が利用者の導入と調査を助けるか。
