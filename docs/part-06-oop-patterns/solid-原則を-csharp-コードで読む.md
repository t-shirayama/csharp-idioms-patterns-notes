# SOLID 原則を C# コードで読む

## 概要

SOLID は、変更に強いオブジェクト指向設計を考えるための原則です。C# では interface、継承、DI、nullable、record などの機能と組み合わせて現れますが、原則を機械的に適用すると抽象化が増えすぎます。

## 使いどころ

- SRP: クラスやメソッドの変更理由を一つに近づける。
- OCP: 既存コードの修正より、新しい実装の追加で変更できる形にする。
- LSP: 派生型や実装型が、呼び出し側の期待を壊さないようにする。
- ISP: 呼び出し側が不要なメンバーに依存しない interface に分ける。
- DIP: 上位方針を下位の詳細実装から切り離す。
- 原則を適用する前に、実際に起きている変更理由やテストの痛点を確認する。

```csharp
public interface IReceiptSender
{
    Task SendAsync(Receipt receipt, CancellationToken cancellationToken);
}

public sealed class CheckoutUseCase
{
    private readonly IReceiptSender _receiptSender;

    public CheckoutUseCase(IReceiptSender receiptSender)
    {
        _receiptSender = receiptSender;
    }

    public Task CompleteAsync(Receipt receipt, CancellationToken cancellationToken) =>
        _receiptSender.SendAsync(receipt, cancellationToken);
}
```

## 判断基準

SOLID はチェックリストとして便利ですが、すべての原則を最大化すると小さなクラスと interface が増えすぎます。まずは変更頻度が高い箇所、外部 I/O と業務判断が混ざっている箇所、テストが書きにくい箇所に絞って見ると効果が出やすくなります。

- SRP は「短いクラス」ではなく「変更理由が混ざっていない」ことを見る。
- OCP は「一切変更しない」ではなく「既存の安定した方針を壊さず拡張できる」ことを見る。
- LSP は継承だけでなく、interface 実装にも当てはまる。
- ISP は `IReadOnly...` や用途別 interface で依存を小さくする。
- DIP は DI コンテナではなく、上位方針が詳細に引きずられないことを見る。

```csharp
public interface IReceiptReader
{
    Task<Receipt?> FindAsync(Guid id, CancellationToken cancellationToken);
}

public interface IReceiptWriter
{
    Task SaveAsync(Receipt receipt, CancellationToken cancellationToken);
}
```

## 避けたい書き方

- SRP を「1 クラス 1 メソッド」と誤解し、関連する処理までばらばらにする。
- OCP のためと言いながら、将来使うか分からない拡張ポイントを先回りして作る。
- LSP を無視して、派生型で `NotSupportedException` を投げる実装を許す。
- DIP を interface 量産と捉え、依存関係の意味を説明できない。

```csharp
// 避けたい例: 契約上は保存できるのに、実装によって突然できない。
public sealed class ReadOnlyReceiptRepository : IReceiptRepository
{
    public Task SaveAsync(Receipt receipt, CancellationToken cancellationToken) =>
        throw new NotSupportedException();
}
```

## よくある失敗

- SRP の名目で処理を分けた結果、1 つの仕様変更で多くのファイルを横断する。
- OCP を意識しすぎて、実際には使われない Strategy や Factory を先に作る。
- ISP を無視して、読み取りだけの画面が更新系メソッドまで持つ repository に依存する。
- DIP を導入したが、実装型名や DB の都合が interface 名やメソッド名に漏れている。

## レビュー観点

- 変更理由が異なる処理が同じクラスに混ざっていないか。
- 新しい要件が来たとき、どのファイルを触るか説明できるか。
- 抽象化によって呼び出し側の意図が読みやすくなっているか。
- 原則に従うことで、テストや保守が実際に楽になっているか。
- 原則のために増えた型が、名前だけで役割を説明できるか。

## Part 6 のレビュー用チェックリスト

- 状態は必要以上に公開されていないか。
- 継承、interface、DI、設計パターンの導入理由を一文で説明できるか。
- 抽象化が業務上の変更点に対応しているか。
- DTO、Entity、Value Object の境界が曖昧になっていないか。
- テストダブルを使うための境界が、本番コードでも自然な責務になっているか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: SRP/OCP/LSP/ISP/DIP それぞれに短い「避けたい例」と「改善例」

> 対応元: P1 / SRP/OCP/LSP/ISP/DIP それぞれに短い「避けたい例」と「改善例」を追加する。

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
