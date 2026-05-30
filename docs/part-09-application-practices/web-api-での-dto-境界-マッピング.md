# Web API での DTO、境界、マッピング

## 概要

Web API の DTO は、外部との契約です。Domain Model や Entity を直接 request / response に使うと、内部構造の変更が API 互換性に影響しやすくなります。

request DTO は入力の形、response DTO は公開する情報を表します。Domain Model への変換は境界で行い、公開してよい項目、互換性を維持したい項目、将来消したい項目を意識して設計します。

```csharp
public sealed record OrderResponse(
    Guid Id,
    string Status,
    DateTimeOffset OrderedAt);

public static OrderResponse ToResponse(Order order)
{
    return new OrderResponse(
        order.Id,
        order.Status.ToString(),
        order.OrderedAt);
}
```

## 使いどころ

- API の request / response を明示し、内部モデルから独立させたい。
- レスポンスから内部 ID、監査情報、不要なナビゲーションプロパティを隠したい。
- API バージョニングや後方互換性を意識する必要がある。

## DTO の設計基準

DTO は「いまの Entity に似ている形」ではなく、クライアントとの契約として設計します。

- request DTO と response DTO を分ける。
- 作成、更新、検索など操作ごとに DTO を分ける。
- optional と required の意味を nullable reference types で表す。
- 金額、時刻、状態などは表現形式を明確にする。
- 互換性のために残す項目には、廃止予定の扱いを決める。

```csharp
public sealed record UpdateCustomerRequest(
    string DisplayName,
    string? PhoneNumber);

public sealed record CustomerResponse(
    Guid Id,
    string DisplayName,
    string? PhoneNumber,
    DateTimeOffset UpdatedAt);
```

更新 API では、`null` が「未指定」なのか「値を消す」なのかが曖昧になりやすいです。PATCH では専用の patch document や明示的な optional 型を検討し、PUT では全置換であることを契約にします。

## マッピングの置き場所

マッピングは Controller に数行で収まるならそこでよい場合もあります。複数 endpoint で使う、変換ルールに判断がある、テストしたい場合は拡張メソッドや mapper に切り出します。

```csharp
public static class OrderMappings
{
    public static OrderResponse ToResponse(this Order order)
    {
        return new OrderResponse(
            order.Id,
            order.Status.ToString(),
            order.OrderedAt);
    }
}
```

AutoMapper などの自動マッピングは、単純な形の変換には有効です。ただし、公開項目の選別、権限による出し分け、状態名の変換など、レビューしたい判断は明示的なコードにした方が安全です。

## API 境界の落とし穴

Entity を直接受け取ると、クライアントが更新できてはいけないプロパティまで送れてしまうことがあります。これは over-posting と呼ばれ、権限や監査情報の破壊につながります。

```csharp
// よい例: クライアントが変更できる値だけを request DTO に持たせる。
public sealed record ChangeDisplayNameRequest(string DisplayName);
```

レスポンスも同様に、内部の enum 名、DB の主キー、論理削除フラグ、監査用項目をそのまま公開してよいとは限りません。公開した項目はクライアントに依存されるため、削除や意味変更には移行期間が必要になります。

## 避けたい書き方

- EF Core の Entity をそのまま JSON として返す。
- request DTO をそのまま Entity として保存し、入力可能な項目を過剰に広げる。
- 自動マッピングに任せきりで、公開項目の意図をレビューできなくする。
- request と response で同じ DTO を使い回し、不要な入力や情報漏えいを招く。
- 日時をローカル時刻の文字列で返し、タイムゾーンの意味を曖昧にする。

```csharp
// 避けたい例: 内部構造やナビゲーションプロパティまで公開される可能性がある。
return Ok(orderEntity);
```

```csharp
// 避けたい例: IsAdmin をクライアントが送れてしまう。
public sealed class UserEntity
{
    public string DisplayName { get; set; } = "";
    public bool IsAdmin { get; set; }
}
```

## レビュー観点

- request DTO と response DTO が用途別に分かれているか。
- Domain Model や Entity を外部契約として直接公開していないか。
- マッピングで secret、内部状態、不要な関連データを落としているか。
- 既存クライアントを壊す変更になっていないか。
- nullable、既定値、日時、enum、金額の表現が API 契約として明確か。
- over-posting や情報漏えいにつながるプロパティが request / response に含まれていないか。
- 自動マッピングの設定が、レビュー可能な粒度でテストされているか。

---

[Part README に戻る](README.md)


