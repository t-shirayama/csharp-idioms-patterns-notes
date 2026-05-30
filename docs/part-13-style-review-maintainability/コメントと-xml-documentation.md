# コメントと XML documentation

## 概要

コメントは「コードを日本語で言い換える」ためではなく、コードだけでは残しにくい判断理由、制約、外部仕様、運用上の注意を補うために使います。読み手がコードからすぐ分かることは、コメントよりも命名や構造で表した方が保守しやすくなります。

XML documentation は、public API やライブラリ境界で特に有効です。呼び出し側が IDE 上で仕様を確認できるため、例外、null、単位、スレッド安全性、キャンセルの扱いなどを短く書く価値があります。

```csharp
/// <summary>
/// 指定したユーザーの月次請求額を計算します。
/// </summary>
/// <remarks>
/// 返金済み明細は計算対象に含めません。
/// </remarks>
public Money CalculateMonthlyCharge(UserId userId, YearMonth targetMonth)
{
    return billingCalculator.Calculate(userId, targetMonth);
}
```

## 判断基準

- コードから読める「何をしているか」は、コメントではなく名前や構造で表す。
- コメントには「なぜその判断になったか」「いつ見直すか」「外部制約は何か」を残す。
- public API の documentation は、呼び出し側が実装を読まずに使える最低限の契約を書く。
- コメントが必要なほど複雑な条件や手順は、先に分割や型の導入で表現できないか検討する。
- TODO は担当、期限、判断材料のどれかを添え、永遠に残るメモにしない。

## 使いどころ

- 業務ルールの背景や、仕様書・外部 API・法令などコード外の根拠を残したいとき。
- public API、共通ライブラリ、拡張ポイントなど、利用者が実装を読まない前提の場所。
- 一見すると不自然だが、互換性や運用都合で必要な処理を残すとき。
- 例外、単位、タイムゾーン、culture、キャンセル、スレッド安全性など、呼び出し側の誤用を防ぎたいとき。

## 避けたい書き方

- コードをそのまま説明するコメントを増やす。
- 実装変更後に古いコメントを残し、コードと説明が矛盾した状態にする。
- 「暫定対応」「あとで直す」だけを書き、期限、理由、判断先を残さない。
- public API の XML documentation で正常系だけを書き、例外や null の扱いを隠す。
- コメントで設計上の違和感を隠し、名前や責務分割の改善を先送りする。

```csharp
// 避けたい例: コメントが処理内容を言い換えているだけ。
// ユーザー名をトリムする
var userName = request.UserName.Trim();
```

```csharp
// 外部決済 API は空白だけの氏名を拒否するため、送信前に正規化する。
var payerName = request.UserName.Trim();
```

```csharp
/// <summary>
/// 指定した注文をキャンセルします。
/// </summary>
/// <exception cref="InvalidOperationException">
/// 出荷済みの注文をキャンセルしようとした場合に発生します。
/// </exception>
public Task CancelAsync(OrderId orderId, CancellationToken cancellationToken)
{
    return orderCancellation.CancelAsync(orderId, cancellationToken);
}
```

## レビュー観点

- コメントは「なぜそうしているか」や「何に注意すべきか」を補っているか。
- コメントとコードが矛盾していないか。古い仕様の説明が残っていないか。
- public API の XML documentation は、呼び出し側が判断に必要な情報を含んでいるか。
- コメントで補う前に、命名、型、メソッド分割で表現できないかを検討しているか。
- TODO や FIXME が、放置されても困らない形ではなく、追跡可能な判断材料になっているか。
- documentation の例外や null の説明が、実際の実装や nullable annotation と一致しているか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: <param>

> 対応元: P1 / `<param>`、`<returns>`、`<exception>`、`<remarks>`、`<inheritdoc>`、`<example>` の使い分け、CS1591 の扱いを追加する。

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
