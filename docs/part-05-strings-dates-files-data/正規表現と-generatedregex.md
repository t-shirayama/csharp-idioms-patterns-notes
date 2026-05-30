# 正規表現と GeneratedRegex

## 概要

正規表現は、単純なパターン抽出や置換には強力です。
一方で、複雑な構文解析、業務ルール、入れ子構造を無理に表現すると、保守しにくくなります。

正規表現は「文字列の形」を見る道具であり、「業務的に正しいか」を全て判断する道具ではない。形式チェックと業務ルールを分けると、エラーメッセージ、テスト、仕様変更が扱いやすくなる。

## 使いどころ

- 形式が安定しているコード、ID、ログ行の一部抽出に使う。
- 繰り返し使う正規表現は `GeneratedRegex` で定義する。
- 単純な前方一致や部分一致なら `StartsWith`、`Contains` で足りるか確認する。

```csharp
private static partial class OrderPatterns
{
    [GeneratedRegex(@"^ORD-\d{8}$", RegexOptions.CultureInvariant)]
    public static partial Regex OrderId();
}

var isValid = OrderPatterns.OrderId().IsMatch(orderId);
```

抽出では、名前付き group を使うと呼び出し側の意図が読みやすい。

```csharp
private static partial class LogPatterns
{
    [GeneratedRegex(
        @"^(?<level>INFO|WARN|ERROR):(?<message>.+)$",
        RegexOptions.CultureInvariant)]
    public static partial Regex LogLine();
}

var match = LogPatterns.LogLine().Match(line);

if (match.Success)
{
    var level = match.Groups["level"].Value;
    var message = match.Groups["message"].Value.Trim();
}
```

## 判断基準

- `StartsWith`、`EndsWith`、`Contains`、`IndexOf` で表せるなら、まず通常の文字列 API を使う。
- 同じ pattern を複数回使うなら `GeneratedRegex` で名前を付ける。
- 外部入力に適用する pattern は、タイムアウトや複雑度を意識する。
- 正規表現は形式チェックまでに留め、DB 照合や重複確認などの業務ルールは別にする。
- パターンが長くなるなら、名前付き group、コメント、テストで意図を補う。

## 避けたい書き方

- パターン文字列を各所に重複して書く。
- ユーザー入力を含む正規表現をそのまま実行する。
- CSV、JSON、HTML のような構造化データを正規表現だけで処理する。
- タイムアウトなしで複雑な正規表現を外部入力に適用する。

```csharp
// 避けたい: 何を検証しているか読み取りにくく、再利用もしにくい
if (Regex.IsMatch(value, "^[A-Z0-9_-]{1,64}$"))
{
    Save(value);
}
```

```csharp
// 良い例: 何の形式か名前で分かる
if (UserNamePatterns.SafeUserName().IsMatch(value))
{
    Save(value);
}
```

```csharp
private static partial class UserNamePatterns
{
    [GeneratedRegex(
        @"^[A-Z0-9_-]{1,64}$",
        RegexOptions.CultureInvariant | RegexOptions.IgnoreCase,
        matchTimeoutMilliseconds: 200)]
    public static partial Regex SafeUserName();
}
```

ユーザー入力を pattern の一部にする必要がある場合は、`Regex.Escape` を使う。それでも複雑な pattern と組み合わせると読みにくくなるため、通常の文字列検索で足りないかを先に確認する。

```csharp
var keywordPattern = Regex.Escape(keyword);
var regex = new Regex(keywordPattern, RegexOptions.CultureInvariant, TimeSpan.FromMilliseconds(200));
```

## 落とし穴

- `^` と `$` は options によって行頭・行末の意味が変わる。文字列全体の一致なら `\A` と `\z` も検討する。
- `.` は改行に一致しない。`Singleline` を付けると意味が変わる。
- `IgnoreCase` は culture の影響を受ける場合がある。機械的な形式では `CultureInvariant` を付ける。
- backtracking が増える pattern は、外部入力で ReDoS の原因になる。
- メールアドレスや URL の完全検証を巨大な正規表現で抱え込むと、保守が難しくなる。

## テストしやすさ

正規表現は、通る例より落ちる例が重要。最小長、最大長、1文字超過、許可されない記号、大小文字、前後の余計な文字、空文字を用意する。抽出 pattern では、group 名と値をテストで固定する。

## レビュー観点

- 正規表現を使う理由が、通常の文字列 API より明確か。
- パターン名やメソッド名で意図が読めるか。
- 外部入力に対してタイムアウトや複雑度のリスクを考慮しているか。
- 境界値、失敗例、大小文字、culture のテストがあるか。
- pattern が複数箇所に重複していないか。
- 正規表現で構造化データや複雑な業務ルールまで処理していないか。
- ユーザー入力を pattern に埋め込む場合、`Regex.Escape` やタイムアウトを使っているか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: RegexOptions.NonBacktracking

> 対応元: P1 / `RegexOptions.NonBacktracking`、`\A` / `\z`、timeout の共通方針、ReDoS テスト例を追加する。

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
