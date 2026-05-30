# DateTime、DateOnly、TimeOnly、DateTimeOffset

## 概要

日付時刻は「日付だけ」「時刻だけ」「絶対時刻」「ローカル表示」のどれを扱っているかを分けます。
`DateTime` だけで済ませると、意図しないタイムゾーン変換やテストしにくさを招きやすくなります。

実務では、日時の型選びがそのまま仕様の曖昧さを表す。請求日、予約枠、イベント発生時刻、画面表示時刻を同じ型で持つと、境界で変換ミスが起きやすい。

## 使いどころ

- 誕生日、営業日、請求対象日などは `DateOnly` を使う。
- 開店時刻や締切時刻のように日付を持たない値は `TimeOnly` を使う。
- 外部 API、DB、ログなど絶対時刻を扱う境界では `DateTimeOffset` を優先する。
- 現在時刻は `TimeProvider` などで注入し、テストで固定できるようにする。

```csharp
public sealed class TrialService(TimeProvider timeProvider)
{
    public bool IsExpired(DateTimeOffset expiresAt)
    {
        return timeProvider.GetUtcNow() >= expiresAt;
    }
}
```

## 判断基準

| 扱いたいもの | 候補 | 例 |
| --- | --- | --- |
| 日付だけ | `DateOnly` | 誕生日、対象営業日、請求日 |
| 時刻だけ | `TimeOnly` | 開店時刻、毎日の締切時刻 |
| 絶対時刻 | `DateTimeOffset` | 作成日時、決済日時、ログ時刻 |
| ローカル表示用 | `DateTimeOffset` + `TimeZoneInfo` | ユーザーの画面表示 |
| 既存 API 都合 | `DateTime` | 古いライブラリ、DB 型との互換 |

日付だけの値を UTC に変換すると、タイムゾーンによって前日や翌日にずれることがある。業務日付は時刻ではなく日付として保存、比較する。

```csharp
public bool IsBillingTarget(DateOnly billingDate, DateOnly today)
{
    return billingDate <= today;
}
```

外部境界に出す日時は、offset を含む形式にすると解釈の揺れが減る。

```csharp
var response = new EventResponse(
    EventId: eventId,
    OccurredAt: occurredAt.ToUniversalTime());
```

現在時刻は `TimeProvider` で注入する。小さなサービスでも、期限判定やリトライ判定はテストで時刻を固定できると保守しやすい。

```csharp
public sealed class CouponService(TimeProvider timeProvider)
{
    public bool CanUse(Coupon coupon)
    {
        var now = timeProvider.GetUtcNow();
        return coupon.StartsAt <= now && now < coupon.ExpiresAt;
    }
}
```

## 避けたい書き方

- `DateTime.Now` を業務ロジックに直接書く。
- 日付だけの値を `DateTime` の 0 時 0 分として扱い続ける。
- `DateTime.Kind` の意味を確認せずに変換する。

```csharp
// 避けたい: テストで現在時刻を固定しにくい
if (DateTime.Now > campaign.EndAt)
{
    CloseCampaign();
}
```

```csharp
// 避けたい例: DateTimeKind.Unspecified の意味が曖昧なまま変換している
var local = DateTime.Parse(input);
var utc = local.ToUniversalTime();
```

```csharp
// 良い例: 入力形式とタイムゾーンを境界で決める
var parsed = DateTimeOffset.Parse(
    input,
    CultureInfo.InvariantCulture,
    DateTimeStyles.AssumeUniversal);
```

`DateTimeOffset` は「タイムゾーン名」を保持する型ではない。offset は `+09:00` のような差分であり、夏時間ルールや地域名は持たない。地域のルールが必要なら `TimeZoneInfo` と組み合わせる。

## テストしやすさ

日時ロジックのテストでは、境界ちょうど、1 tick 前後、月末、うるう年、日付が変わる時刻を入れる。`DateTime.Now` を直接呼ぶコードはテストが不安定になりやすいため、`TimeProvider` や clock interface を通して固定する。

## レビュー観点

- 対象が日付だけなら `DateOnly`、絶対時刻なら `DateTimeOffset` になっているか。
- 現在時刻への依存が注入可能か。
- DB や JSON で保存する形式が明確か。
- `DateTime.Now`、`DateTime.UtcNow`、`DateTimeOffset.UtcNow` が散らばっていないか。
- `DateTimeKind.Unspecified` の値を暗黙に UTC / local として扱っていないか。
- `DateTimeOffset` にタイムゾーン名まで入っていると誤解していないか。
- 日付だけの仕様に、0 時 0 分の `DateTime` を使って境界バグを作っていないか。

---

[Part README に戻る](README.md)


