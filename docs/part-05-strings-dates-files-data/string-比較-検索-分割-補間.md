# string 比較、検索、分割、補間

## 概要

`string` は不変 (immutable) な参照型です。
比較や検索では、既定の挙動に任せるのではなく、大小文字や culture をどう扱うかを明示します。

文字列処理の不具合は、開発者の環境では再現しないことが多い。入力の前後空白、大小文字、全角半角、culture、区切り文字、ログに出してよい情報を、コード上で判断できる形にしておく。

## 使いどころ

- ID、コード、キーなどの機械的な文字列は `StringComparison.Ordinal` または `OrdinalIgnoreCase` を使う。
- 画面表示やユーザー向けの並び替えでは culture を意識する。
- 複数値の組み立ては `string.Join`、読みやすいメッセージは文字列補間を使う。

```csharp
var isAdmin = role.Equals("admin", StringComparison.OrdinalIgnoreCase);

var path = string.Join("/", segments.Where(s => !string.IsNullOrWhiteSpace(s)));

var message = $"User {userId} failed to login at {failedAt:O}.";
```

`Equals`、`Contains`、`StartsWith`、`EndsWith` では比較ルールを明示する。C# / .NET のバージョンによって overload が増えているため、使える場合は `StringComparison` 付きの overload を選ぶ。

```csharp
if (fileName.EndsWith(".json", StringComparison.OrdinalIgnoreCase))
{
    ImportJson(fileName);
}
```

ユーザー向けの並び替えは culture に寄せる。機械的なキーの並び替えとは分けて考える。

```csharp
var comparer = StringComparer.Create(
    CultureInfo.GetCultureInfo("ja-JP"),
    ignoreCase: false);

var displayNames = users
    .Select(user => user.DisplayName)
    .OrderBy(name => name, comparer)
    .ToList();
```

分割は簡単に見えるが、CSV や引用符つきの形式には向かない。単純な区切り文字だけを扱うときも、空要素と空白の扱いを決める。

```csharp
var tags = input
    .Split(',', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries)
    .Distinct(StringComparer.OrdinalIgnoreCase)
    .ToList();
```

## 判断基準

- ID、コード、ファイル拡張子、HTTP ヘッダー名などは `Ordinal` 系で比較する。
- 人間に見せる表示名や検索語は、対象 culture と仕様を確認する。
- `null`、空文字、空白だけの文字列を同じ扱いにするか、別扱いにするかを先に決める。
- 単純な区切り文字なら `Split`、CSV やエスケープを含む形式なら専用 parser を使う。
- ログや例外メッセージでは、個人情報、トークン、パスワードを含めない。

## 避けたい書き方

- `ToLower()` や `ToUpper()` してから比較する。
- `Contains("x")` のように比較ルールを暗黙にする。
- `Split(',')` の結果に空白や空要素が混じる前提を放置する。

```csharp
// 避けたい: culture と余分な allocation が混ざる
if (role.ToLower() == "admin")
{
    GrantAccess();
}
```

```csharp
// 良い例: 比較意図を明示する
if (string.Equals(role, "admin", StringComparison.OrdinalIgnoreCase))
{
    GrantAccess();
}
```

文字列補間は読みやすいが、culture 依存の値をログや外部連携へ出すときは形式を固定する。日時は round-trip 形式、数値は invariant culture を検討する。

```csharp
var line = string.Create(
    CultureInfo.InvariantCulture,
    $"amount={amount:F2}, createdAt={createdAt:O}");
```

## テストしやすさ

文字列処理のテストでは、正常系だけでなく、前後空白、空文字、大小文字違い、想定外の区切り、`null` を入れる。比較ルールを明示しておくと、テスト名も「OrdinalIgnoreCase で一致する」のように仕様として書ける。

## レビュー観点

- 比較に `StringComparison` が指定されているか。
- 入力文字列の前後空白、空文字、null の扱いが明確か。
- 区切り文字で分割する場合、空要素や引用符つき CSV などを単純な `Split` で処理していないか。
- ログや例外メッセージに個人情報や secret を埋め込んでいないか。
- 文字列補間で日時や数値を出すとき、保存形式や外部連携形式が culture に依存していないか。
- 大量ループ内で不要な `ToLower`、`Trim`、連結を繰り返していないか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: Unicode 正規化

> 対応元: P1 / Unicode 正規化、全角半角、結合文字、`Length` と見た目の文字数の違い、検索・ID 正規化の境界を追加する。

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
