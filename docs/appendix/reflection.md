# reflection

## 概要

reflection は、実行時に型、プロパティ、属性などのメタデータを読むための仕組みです。
柔軟ですが、コンパイル時の安全性が弱く、コストや trimming / AOT との相性にも注意が必要です。

reflection は「通常のコードでは表現しづらい汎用境界」に閉じると有効です。
一方で、業務ルールの分岐や型ごとの処理をすべて reflection に寄せると、リネームに弱く、テストで失敗を見つけにくくなります。

```csharp
var property = typeof(User).GetProperty(nameof(User.DisplayName))
    ?? throw new InvalidOperationException("Property not found.");

var displayName = property.GetCustomAttribute<DisplayNameAttribute>()?.DisplayName
    ?? property.Name;
```

## 判断基準

- フレームワーク、シリアライザー、DI、validation など型を横断する処理に限定する。
- 型が決まっている業務処理では、interface、generic、pattern matching を先に検討する。
- hot path で使う場合は測定し、`MemberInfo` や delegate をキャッシュする。
- trimming / NativeAOT 対応が必要なら、属性や source generator で静的に表現できないか検討する。
- private member へのアクセスは最後の手段とし、変更耐性と権限の問題をレビューする。

## 使いどころ

- 属性を読んで、表示名、検証ルール、シリアライズ対象などを決める。
- 汎用ライブラリやフレームワーク境界で、型情報から処理を組み立てる。
- 一度調べた `PropertyInfo` や delegate をキャッシュして再利用する。
- plugin や拡張ポイントで、外部 assembly の型を発見する。

## よい例

属性読み取りを一箇所に閉じ、呼び出し側は通常の文字列として扱います。

```csharp
static string GetDisplayName<T>(Expression<Func<T, object?>> propertySelector)
{
    if (propertySelector.Body is not MemberExpression member)
    {
        throw new ArgumentException("Property selector is required.", nameof(propertySelector));
    }

    return member.Member.GetCustomAttribute<DisplayNameAttribute>()?.DisplayName
        ?? member.Member.Name;
}
```

## 避けたい書き方

- アプリケーションの通常の分岐を reflection で置き換える。
- 文字列でプロパティ名を指定し、リネームで壊れる状態にする。
- hot path で毎回 `GetProperties()` や `GetValue()` を繰り返す。
- 例外を握りつぶして、メンバーが見つからない状態を既定値で隠す。
- private member 名に依存し、依存先の内部実装変更で壊れる。

```csharp
// 避けたい例: リネームに弱く、失敗しても compile 時に分からない。
var value = typeof(User).GetProperty("DisplayName")!.GetValue(user);
```

```csharp
// 避けたい例: 型ごとの分岐なら通常の C# で書いた方が追いやすい。
var method = handler.GetType().GetMethod("Handle");
method?.Invoke(handler, new object[] { command });
```

## テストしやすくする

reflection を使う処理は、入力型と期待 metadata の組み合わせを小さな fixture でテストします。
missing member、属性なし、継承、nullable、generic type など、現場で壊れやすい形を最初に押さえます。

```csharp
public sealed class SampleModel
{
    [DisplayName("表示名")]
    public string Name { get; init; } = "";
}
```

## レビュー観点

- `nameof` や属性型を使い、文字列依存を減らしているか。
- reflection が必要な境界に閉じていて、通常の業務ロジックへ広がっていないか。
- 高頻度に呼ばれる処理では測定し、必要ならキャッシュしているか。
- trimming / NativeAOT 対応が必要なアプリで、実行時に型が消える前提を確認しているか。
- `BindingFlags` の指定が広すぎず、意図しない member を拾わないか。
- 失敗時の例外が、型名、member 名、期待条件を含んで調査しやすいか。

---

[Appendix README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: trimming / NativeAOT 向けの DynamicallyAccess…

> 対応元: P1 / trimming / NativeAOT 向けの `DynamicallyAccessedMembers`、`DynamicDependency`、`RequiresUnreferencedCode`、`NullabilityInfoContext`、`AssemblyLoadContext`、`TargetInvocationException` の扱いを追加する。

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
