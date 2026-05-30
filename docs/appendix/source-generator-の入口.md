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


