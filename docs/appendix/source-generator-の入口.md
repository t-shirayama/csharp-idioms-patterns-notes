# source generator の入口

## 概要

source generator は、コンパイル時に C# コードを生成する仕組みです。
reflection を減らす、定型コードを自動生成する、利用者に analyzer と組み合わせてフィードバックする、といった用途があります。

source generator は強力ですが、導入するとビルド、IDE、デバッグ、エラーメッセージの体験にも責任を持つことになります。
実行時の複雑さをコンパイル時に移す道具なので、生成されるコードが利用者にとって追える形かを重視します。

## 使いどころ

- 属性や型定義から、シリアライズ、マッピング、DI 登録などの定型コードを生成する。
- reflection の実行時コストや trimming 依存を減らしたい。
- ライブラリ利用者のコードに対して、コンパイル時に診断を出したい。
- コンパイル時に分かる不整合を、実行時例外ではなく診断として返したい。

```csharp
[GenerateMapper]
public partial class UserMapper
{
    public partial UserDto ToDto(User user);
}
```

## 判断基準

- 生成対象が十分に反復的で、手書きすると漏れやすいか。
- reflection の削減、AOT 対応、起動時間短縮など、実行時の利点が説明できるか。
- 利用者が生成コードを見なくても直せる診断を出せるか。
- generator 自体の保守コストが、削減できる定型コード量に見合うか。
- 入力が syntax tree、semantic model、additional file のどれなのかを明確にする。

## よい例

利用者側の API は `partial` と属性にとどめ、生成コードの存在を意識しすぎない形にします。

```csharp
[OptionsValidator]
public sealed partial class PaymentOptionsValidator
{
}

public sealed class PaymentOptions
{
    [Required]
    public string Endpoint { get; init; } = "";
}
```

生成側では、同じ入力から同じ出力が出るようにします。

```csharp
// よい例: 出力名が型名から安定して決まる。
context.AddSource($"{typeName}.g.cs", generatedSource);
```

## 避けたい書き方

- 数行の手書きで済む処理まで generator にする。
- 生成コードを確認する手段を用意しない。
- ビルド時の入力が不安定で、差分のたびに大量の生成コードが変わる。
- 現在時刻、乱数、環境変数などで生成結果が変わる。
- エラー時に `throw` してビルドを壊し、利用者に修正箇所を示さない。

```csharp
// 避けたい例: 生成結果がビルドごとに変わり、差分確認が難しい。
var source = $"// Generated at {DateTimeOffset.Now}\n{body}";
```

## テストしやすくする

generator は入力コード、期待される診断、生成コードの要点をテストします。
snapshot に頼りすぎると差分が大きくなるため、重要な API、属性、nullability、診断 ID を中心に確認します。

```csharp
const string Source = """
[GenerateMapper]
public partial class UserMapper
{
    public partial UserDto ToDto(User user);
}
""";
```

## レビュー観点

- 導入理由が、定型削減、実行時コスト削減、AOT 対応など明確か。
- incremental generator を使い、変更された入力だけを処理できる設計か。
- 生成コードが読みやすく、デバッグ時に追えるか。
- analyzer の診断メッセージが、利用者に次の修正行動を示しているか。
- 生成コードが nullable context、namespace、accessibility を正しく扱っているか。
- 生成対象の名前衝突、partial 条件、generic 型、nested type を考慮しているか。

---

[Appendix README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: incremental generator の処理分解

> 対応元: P1 / incremental generator の処理分解、`AdditionalTextsProvider`、`AnalyzerConfigOptions`、diagnostic location、`EmitCompilerGeneratedFiles`、generator テスト、NuGet packaging を追加する。

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
