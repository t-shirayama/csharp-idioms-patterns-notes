# タイムゾーンと UTC

## 概要

サーバー内部や保存形式では UTC を基本にし、ユーザーに見せる直前でローカル時刻へ変換します。
ただし、日付だけの業務ルールでは UTC 変換がかえって誤りになることがあります。

タイムゾーンの問題は「UTC にすれば全部解決」ではない。発生した瞬間、保存する瞬間、集計する営業日、ユーザーに表示する地域を分けて考える。特に夏時間のある地域では、存在しない時刻や 2 回現れる時刻がある。

## 使いどころ

- 作成日時、更新日時、イベント発生時刻は UTC で保存する。
- ユーザー表示や帳票出力では対象ユーザーまたは業務拠点のタイムゾーンへ変換する。
- 「日本時間の 2026-04-01」のような業務日付は、時刻ではなく日付として扱う。

```csharp
var utcNow = timeProvider.GetUtcNow();
var tokyo = TimeZoneInfo.FindSystemTimeZoneById("Tokyo Standard Time");
var localTime = TimeZoneInfo.ConvertTime(utcNow, tokyo);
```

## 判断基準

- 保存するイベント時刻は UTC の `DateTimeOffset` を基本にする。
- 表示ではユーザー、店舗、契約、帳票など、どのタイムゾーンを使うかを明示する。
- 集計単位が「現地日付」なら、UTC 日付で丸めない。
- タイムゾーン ID は設定値やユーザープロファイルとして管理し、コードに散らさない。
- Windows と Linux の両方で動くサービスでは、タイムゾーン ID の互換性を確認する。

UTC 時刻からユーザーの現地日付を求める場合は、先にタイムゾーン変換してから日付を取り出す。

```csharp
var userZone = TimeZoneInfo.FindSystemTimeZoneById(user.TimeZoneId);
var userTime = TimeZoneInfo.ConvertTime(order.CreatedAt, userZone);
var userDate = DateOnly.FromDateTime(userTime.DateTime);
```

「日本時間の今日」のような業務日付は、基準タイムゾーンを明示して作る。

```csharp
var now = timeProvider.GetUtcNow();
var zone = TimeZoneInfo.FindSystemTimeZoneById("Tokyo Standard Time");
var todayInTokyo = DateOnly.FromDateTime(
    TimeZoneInfo.ConvertTime(now, zone).DateTime);
```

## 避けたい書き方

- サーバーのローカルタイムゾーンに依存する。
- UTC の `DateTimeOffset` を日付に丸めてからユーザーのタイムゾーンへ変換する。
- タイムゾーン ID を画面や DB に説明なしで埋め込む。

```csharp
// 避けたい: 実行環境のタイムゾーンで結果が変わる
var today = DateTime.Today;
```

```csharp
// 避けたい例: UTC 日付で丸めると、ユーザーの現地日付とずれる
var createdDate = order.CreatedAt.UtcDateTime.Date;
```

```csharp
// 良い例: 表示・集計対象のタイムゾーンに変換してから日付化する
var localCreatedAt = TimeZoneInfo.ConvertTime(order.CreatedAt, userZone);
var createdDate = DateOnly.FromDateTime(localCreatedAt.DateTime);
```

夏時間がある地域では、ローカル時刻から UTC に戻すときに曖昧さが出る。ユーザー入力がローカル時刻なら、`TimeZoneInfo.IsInvalidTime` や `IsAmbiguousTime` を使って扱いを決める。

```csharp
if (timeZone.IsInvalidTime(localDateTime))
{
    return ValidationError("This local time does not exist.");
}
```

## テストしやすさ

タイムゾーンのテストは、日本だけで完結させない。夏時間がある地域、UTC から日付が前後する地域、日付境界の直前直後を入れる。OS 差があるサービスでは、タイムゾーン ID を直接散らさず、変換サービスを薄く切り出すとテストしやすい。

## レビュー観点

- 保存、計算、表示のどこでタイムゾーン変換しているかが分かるか。
- 日付だけの仕様に UTC 変換を持ち込んでいないか。
- 夏時間のある地域で境界時刻のテストがあるか。
- Windows と Linux でタイムゾーン ID の扱いに差が出ないか。
- サーバーの `DateTime.Today` や `TimeZoneInfo.Local` に業務仕様が依存していないか。
- UTC に丸めてから現地日付へ変換していないか。
- ユーザー入力のローカル時刻に対して、存在しない時刻と曖昧な時刻の扱いが決まっているか。

---

[Part README に戻る](README.md)


