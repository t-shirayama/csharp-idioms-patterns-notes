# CI、analyzers、formatters、tests

## 概要

CI は「テストを走らせる場所」だけではありません。formatter、analyzer、nullable warning、test、coverage、package validation などを組み合わせ、品質低下を早めに止めるための仕組みです。

ただし、品質ゲートを強くしすぎると、既存負債で開発が止まります。重要なのは、どの警告を必須にするか、どこから段階導入するか、ローカルで再現できるか、失敗時に直し方が分かるかです。

## 判断基準

- formatter は議論を減らすために自動化する。
- analyzer は correctness、security、maintainability に効くものから error 化する。
- nullable warning は新規コードから厳しくし、既存コードは範囲を区切って改善する。
- test は速い単体テストと遅い結合テストを分ける。
- coverage は数字だけでなく、重要な判断と失敗時の挙動を守るために使う。

## 使いどころ

チーム開発、OSS、社内共通ライブラリ、Web API、バッチ、運用ツールなど、複数人で継続的に変更する C# プロジェクトで使います。特にリファクタリング、依存更新、SDK 更新、nullable 有効化では CI の品質ゲートが安全網になります。

## 良い例

プロジェクト側で nullable と analyzer の方針を明示します。

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>

  <PropertyGroup>
    <WarningsNotAsErrors>CS1591</WarningsNotAsErrors>
  </PropertyGroup>
</Project>
```

すべてを一律に error にするのではなく、例外にする警告も明示します。

## 避けたい書き方

CI だけで通る特殊なコマンドにすると、失敗時に開発者が手元で再現できません。

```yaml
- run: dotnet test --configuration Release
```

この一行だけでは restore、format、build、analyzer、test category、coverage の方針が見えません。

## 改善例

品質ゲートを段階に分け、ローカルでも同じ順序で実行できるようにします。

```yaml
- name: Restore
  run: dotnet restore

- name: Format check
  run: dotnet format --verify-no-changes --no-restore

- name: Build
  run: dotnet build --configuration Release --no-restore

- name: Test
  run: dotnet test --configuration Release --no-build --logger trx
```

必要なら `Directory.Build.props` や `Makefile`、PowerShell script にまとめ、CI とローカルの差を減らします。

## レビュー観点

- CI の失敗が、開発者の手元で再現できるコマンドになっているか。
- formatter と analyzer の責務が分かれ、スタイル議論を PR で繰り返していないか。
- `TreatWarningsAsErrors` の導入範囲が、既存負債とチームの対応能力に合っているか。
- flaky test を放置せず、隔離、修正、リトライ条件が明確か。
- coverage gate が、意味のないテスト追加や重要分岐の見落としを招いていないか。
- package や Docker image などの成果物を作る場合、ビルドだけでなく検証も行っているか。
