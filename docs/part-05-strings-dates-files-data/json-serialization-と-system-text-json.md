# JSON serialization と System.Text.Json

## 概要

`System.Text.Json` は標準的な JSON 処理として多くのアプリケーションで十分に使えます。
重要なのは、DTO、命名規則、null、enum、互換性を API 境界の契約として扱うことです。

JSON は単なる変換結果ではなく、外部との契約です。一度公開したプロパティ名、enum 値、null の扱い、日時形式は、呼び出し側のコードに組み込まれる。内部モデルの都合で安易に変えない。

## 使いどころ

- 外部公開 API や保存形式では、domain model ではなく DTO を用意する。
- JSON の命名規則は `JsonSerializerOptions` で統一する。
- enum の文字列化や null の扱いは、互換性への影響を考えて決める。

```csharp
public sealed record UserResponse(
    string Id,
    string DisplayName,
    DateTimeOffset CreatedAt);

var options = new JsonSerializerOptions(JsonSerializerDefaults.Web)
{
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
};

var json = JsonSerializer.Serialize(response, options);
```

`JsonSerializerOptions` はアプリケーション境界で共有する。呼び出し箇所ごとに作ると、命名規則や converter がずれて契約が壊れやすい。

```csharp
public static class JsonOptions
{
    public static readonly JsonSerializerOptions Web = new(JsonSerializerDefaults.Web)
    {
        DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
        Converters = { new JsonStringEnumConverter() }
    };
}
```

DTO では nullable と required を仕様として表す。受け取り側では、欠落、null、未知のプロパティに対する扱いを決める。

```csharp
public sealed record CreateUserRequest
{
    public required string Email { get; init; }
    public string? DisplayName { get; init; }
}
```

## 判断基準

- API やファイル形式の境界では DTO を使い、domain model を直接出さない。
- 命名規則は `JsonSerializerDefaults.Web` や共通 options で統一する。
- enum を数値で出すか文字列で出すかは、互換性と可読性のトレードオフで決める。
- null を省略するか出力するかは、クライアントの解釈に影響する。
- 日時は ISO 8601 / offset 付きの形式を基本にし、独自形式が必要なら converter とテストを用意する。

## 避けたい書き方

- domain model をそのまま JSON に出す。
- 呼び出し箇所ごとに異なる `JsonSerializerOptions` を作る。
- 後方互換性を考えずにプロパティ名や enum 値を変更する。
- 例外を握りつぶして不完全な DTO を返す。

```csharp
// 避けたい: 内部状態まで外部契約に混ざる
var json = JsonSerializer.Serialize(order);
```

```csharp
// 良い例: 外部に出す形を DTO に限定する
var response = new OrderResponse(
    Id: order.Id,
    Status: order.Status.ToString(),
    TotalAmount: order.TotalAmount);

var json = JsonSerializer.Serialize(response, JsonOptions.Web);
```

deserialize では、例外を握りつぶして既定値の DTO を使うと、データ破損を正常系に見せてしまう。入力が不正なら、境界でエラーにする。

```csharp
try
{
    return JsonSerializer.Deserialize<CreateUserRequest>(json, JsonOptions.Web)
        ?? throw new JsonException("Request body is empty.");
}
catch (JsonException ex)
{
    throw new InvalidRequestException("Invalid JSON payload.", ex);
}
```

## 互換性の考え方

- プロパティ追加は比較的安全だが、古いクライアントが未知プロパティを無視できるか確認する。
- プロパティ削除、名前変更、型変更は破壊的変更になりやすい。
- enum 値の追加は、古いクライアントの default 分岐に流れる可能性がある。
- `required` を追加すると、古い送信側が壊れる可能性がある。
- null を省略する変更は、「明示的な null」と「欠落」を区別するクライアントに影響する。

## テストしやすさ

JSON のテストでは、DTO の round-trip だけでなく、実際の JSON 文字列を固定して契約を確認する。未知プロパティ、欠落プロパティ、null、未知 enum、日時形式を入れると、互換性の事故を見つけやすい。

## レビュー観点

- DTO が API やファイル形式の契約として分離されているか。
- required、nullable、既定値の扱いが明確か。
- 古いクライアントが未知のプロパティや欠落したプロパティに耐えられるか。
- custom converter が必要な理由とテストがあるか。
- options が呼び出し箇所ごとに分散していないか。
- enum、日時、decimal の表現が外部仕様として明示されているか。
- deserialize 失敗を握りつぶして、空 DTO や既定値で処理を続けていないか。

---

[Part README に戻る](README.md)


