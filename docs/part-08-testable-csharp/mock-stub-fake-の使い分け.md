# mock / stub / fake の使い分け

## 概要

test double は、テスト対象の外側にある依存を置き換えるための道具です。stub は決まった値を返すもの、fake は軽量な実装を持つもの、mock は呼び出しの有無や引数を検証するものとして考えると整理しやすくなります。spy は呼び出された内容を記録して後で確認する用途です。

## 使いどころ

- stub: 外部サービスのレスポンスや設定値など、入力条件を固定したい。
- fake: in-memory repository など、振る舞いを持つ軽量実装で複数ケースを試したい。
- mock / spy: メール送信、イベント発行など、外部への通知が発生したことを確認したい。

```csharp
public sealed class StubExchangeRateProvider : IExchangeRateProvider
{
    public decimal GetRate(string currency) => currency == "USD" ? 150m : 1m;
}
```

## 使い分けの判断基準

test double は、テストで何を固定したいか、何を観測したいかで選びます。

- 値を返すだけなら stub。
- 小さな状態を持たせたいなら fake。
- 呼び出された事実や引数を確認したいなら mock / spy。
- 本物の制約を確認したいなら integration test。

```csharp
public sealed class SpyMailSender : IMailSender
{
    public List<MailMessage> SentMessages { get; } = new();

    public Task SendAsync(MailMessage message, CancellationToken cancellationToken)
    {
        SentMessages.Add(message);
        return Task.CompletedTask;
    }
}
```

メール送信のように戻り値より副作用が重要な依存では、spy が読みやすいことがあります。mocking library を使う場合も、検証したい引数だけに絞ります。

## 所有している境界を包む

外部ライブラリや SDK を直接 mock すると、ライブラリの実装詳細にテストが引きずられます。自分たちの用途に合わせた小さな interface を作り、その adapter を本番コードで実装します。

```csharp
public interface IPaymentGateway
{
    Task<PaymentResult> AuthorizeAsync(
        PaymentRequest request,
        CancellationToken cancellationToken);
}
```

この interface は「自分たちが必要とする支払い境界」を表します。SDK の全機能を写した大きな interface にすると、結局テストも実装も重くなります。

## fake の限界

in-memory fake は速くて便利ですが、本物の DB や HTTP とは制約が違います。ユニーク制約、transaction、SQL の変換、文字列比較の照合順序、タイムアウトは fake では再現しにくいです。

```csharp
// よい例: 業務ルールのテストでは fake で十分なことがある。
var repository = new FakeOrderRepository();
repository.Add(new OrderBuilder().Build());
```

repository の LINQ や migration を検証したい場合は、fake ではなく実 DB に近い統合テストを持つ方が安全です。

## 避けたい書き方

- 実装の細かい呼び出し順序まで mock で固定し、リファクタリングに弱くする。
- 自分たちが所有していない複雑な外部ライブラリを直接 mock しようとする。
- DB や HTTP の失敗を fake で隠し、本番に近い境界テストをまったく持たない。
- mock の setup が長すぎて、テスト対象の仕様より準備の方が読みにくい。
- mock を使うためだけに、本番コードへ意味の薄い interface を増やす。

```csharp
// 避けたい例: 内部実装の手順を固定しすぎると、同じ仕様の改善でもテストが壊れる。
repositoryMock.Verify(x => x.Begin());
repositoryMock.Verify(x => x.Save(order));
repositoryMock.Verify(x => x.Commit());
```

```csharp
// 避けたい例: 呼び出し回数だけを見て、利用者に見える結果を確認していない。
mailSenderMock.Verify(x => x.SendAsync(It.IsAny<MailMessage>(), default), Times.Once);
```

## レビュー観点

- 検証しているのは外部から見える振る舞いか、内部手順か。
- mock が多すぎて、テスト対象の設計が依存だらけになっていないか。
- 外部 I/O は薄い adapter で包まれ、置き換える境界が明確か。
- fake が本物と異なる制約を持ち、誤った安心を与えていないか。
- test double の名前から、stub、fake、spy の役割が分かるか。
- mock の検証が、仕様として重要な副作用や契約に限られているか。

---

[Part README に戻る](README.md)


