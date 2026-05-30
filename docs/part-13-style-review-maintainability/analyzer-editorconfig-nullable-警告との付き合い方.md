# analyzer、EditorConfig、nullable 警告との付き合い方

## 概要

analyzer、`.editorconfig`、nullable warning は、チームのレビュー観点を機械的に支援する仕組みです。人間のレビューで毎回同じスタイル指摘をするより、ツールに任せられるものはツールに寄せると、本質的な設計や仕様の確認に時間を使えます。

ただし、警告はすべて機械的に消せばよいわけではありません。ルールの意図を理解し、プロジェクトの文脈に合わない場合は、設定変更、局所的な抑制、設計の見直しのどれが適切かを選びます。

```csharp
#nullable enable

public string GetDisplayName(User? user)
{
    if (user is null)
    {
        return "Unknown";
    }

    return user.DisplayName;
}
```

## 判断基準

- analyzer は「レビューで毎回言うこと」を自動化するために使う。
- nullable 警告は、null の可能性を型に戻すためのヒントとして扱う。
- `!` は「ここでは null ではない」と証明できる場合の最後の手段にする。
- ルールを無効化するときは、個人の好みではなくチームの保守性に照らして理由を残す。
- 既存コードへ導入するときは、対象範囲を小さくし、機能変更と警告対応を分ける。

## 使いどころ

- 命名、フォーマット、nullable、不要な using、非同期の誤用などを継続的に検知したいとき。
- 新規プロジェクトで、チームの最低限のスタイルを最初に揃えたいとき。
- 既存プロジェクトに nullable reference types を段階的に導入したいとき。
- CI で警告を見える化し、レビュー前に基本的な問題を落としたいとき。

## 避けたい書き方

- nullable 警告を消すためだけに `!` を付け、実際の null 可能性を隠す。
- analyzer の警告を一括抑制し、どのルールをなぜ無効化したか分からない状態にする。
- `.editorconfig` を個人の好みで頻繁に変え、PR ごとに差分を大きくする。
- 既存コード全体に一度に厳しいルールを適用し、機能変更と大量修正を混ぜる。
- warning をゼロにすること自体が目的になり、設計上の問題を見逃す。

```csharp
// 避けたい例: null の可能性を設計で解決せず、警告だけを消している。
return user!.DisplayName;
```

```csharp
if (user is null)
{
    return DisplayName.Unknown;
}

return DisplayName.From(user.DisplayName);
```

```csharp
// 局所的な抑制は、理由と範囲を小さくする。
#pragma warning disable CA2000
var stream = response.GetStreamOwnedByCaller();
#pragma warning restore CA2000
```

## EditorConfig の考え方

`.editorconfig` は、チームの合意を機械が読める形にしたものです。強制したいルール、提案に留めたいルール、既存コードでは段階導入するルールを分けると、レビューが荒れにくくなります。

```editorconfig
dotnet_style_qualification_for_field = false:suggestion
dotnet_diagnostic.CA1062.severity = warning
dotnet_diagnostic.CA2007.severity = none
```

上のような設定は、なぜその severity にしているかを PR で説明できる状態にしておきます。特に public API、nullable、セキュリティ、リソース解放に関わる警告は、単なるスタイルより重く扱います。

## レビュー観点

- analyzer の警告を、単なる雑音ではなく設計上のヒントとして確認しているか。
- `#pragma warning disable` や `!` の使用箇所に、妥当な理由があるか。
- `.editorconfig` の変更がチーム全体に与える影響を説明できるか。
- nullable の注釈が外部入力、DB、JSON、古い API との境界で正しく表現されているか。
- 新しいルール導入時に、既存修正と機能変更を分けてレビューできる粒度になっているか。
- warning の抑制範囲が最小で、後から削除できる形になっているか。
- CI で失敗させる警告と、IDE 上の提案に留める警告が区別されているか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: .editorconfig の severity 方針

> 対応元: P0 / `.editorconfig` の severity 方針、`TreatWarningsAsErrors`、CI で落とす警告、IDE suggestion に留める警告の分類例を追加する。

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


## TODO対応: nullable attributes の具体例として NotNullWhen

> 対応元: P0 / nullable attributes の具体例として `NotNullWhen`、`MemberNotNull`、`MaybeNull`、`DisallowNull`、`required`、JSON/DB 境界での null 注釈を追加する。

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
