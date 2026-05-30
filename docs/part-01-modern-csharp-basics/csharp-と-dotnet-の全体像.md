# C# と .NET の全体像

## 概要

C# は言語、.NET は実行基盤とライブラリ群です。実務では「どの C# 構文が使えるか」と「どの .NET API が使えるか」が別の話になるため、SDK、Runtime、Target Framework Moniker (TFM) を分けて見る必要があります。

- SDK はビルドやテンプレート作成に使う道具
- Runtime はアプリケーションを実行する環境
- Target Framework はアプリケーションが前提にする API 範囲
- Console、Web API、Worker Service では起動方法や依存注入、ログ、設定の定石が異なる

## 判断基準

実務では、言語バージョン、SDK、Runtime、TFM を別々に確認します。C# の新機能が使えることと、実行環境で新しい .NET API が使えることは同じではありません。ライブラリ開発では利用者の TFM に合わせ、アプリケーション開発では運用環境と CI の SDK を基準にします。

新規開発では「どこで動くか」「どのくらい保守するか」「周辺ライブラリが対応しているか」を先に決めます。既存開発では、最新化そのものよりも、ビルド、テスト、デプロイ、監視が同じ前提で動く状態を優先します。

## 使いどころ

新規プロジェクトでは、まず TFM とアプリケーション種別を決めます。たとえば業務 Web API なら ASP.NET Core、常駐処理なら Worker Service、バッチや検証ツールなら Console が自然です。

既存プロジェクトでは、言語機能の導入前に `.csproj` の `TargetFramework` と `LangVersion`、CI の SDK バージョンを確認します。ローカルだけ新しい SDK で動く状態は、チーム開発では事故の入口になります。

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
</Project>
```

## 良い例

アプリケーションの入口では、標準の Host、設定、ログ、DI に乗せると、環境差分やテスト時の差し替えが扱いやすくなります。小さな Console ツールでも、長く使うなら `ILogger` や `IConfiguration` に寄せる価値があります。

```csharp
var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddHostedService<ImportWorker>();
builder.Services.AddSingleton<IClock, SystemClock>();

await builder.Build().RunAsync();
```

## 避けたい書き方

「C# が新しいから使えるはず」と考えて、ランタイムやターゲットを確認せずに API を追加するのは避けます。構文はコンパイルできても、運用環境の Runtime が古い、または対象 TFM に API がない場合があります。

また、Web API や Worker Service で Console アプリの感覚のまま `Console.WriteLine` や手作りの設定読み込みを広げると、ログ集約、設定差し替え、テストが難しくなります。

```csharp
// 避けたい例: 実行環境に依存する情報を散らばらせる
Console.WriteLine("Started");
var connectionString = Environment.GetEnvironmentVariable("DB_CONNECTION");
```

## 改善例

設定は型に束ね、ログは呼び出し元ではなく処理の境界で文脈を付けます。これにより、本番、検証、ローカルの差分を設定ファイルや環境変数に閉じ込めやすくなります。

```csharp
public sealed class DatabaseOptions
{
    public required string ConnectionString { get; init; }
}

builder.Services
    .AddOptions<DatabaseOptions>()
    .BindConfiguration("Database")
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

## レビュー観点

- `.csproj` の `TargetFramework` は運用環境と合っているか
- SDK バージョンが CI や開発者間で揃うように管理されているか
- アプリケーション種別に合ったログ、設定、DI の仕組みを使っているか
- 言語機能とランタイム機能を混同した説明やコメントがないか
- ライブラリで必要以上に新しい TFM を要求して、利用側の更新を強制していないか
- `global.json`、コンテナ、CI 設定の SDK バージョンが矛盾していないか

---

[Part README に戻る](README.md)


