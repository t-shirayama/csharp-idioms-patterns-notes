# trimming / NativeAOT 互換性

trimming と NativeAOT は、使われないコードを削ることで配布サイズや起動時間を改善します。一方で、reflection、動的ロード、JSON serialization、DI、native interop は「静的に使われている」ことを解析しにくく、実行時だけ失敗することがあります。

## 判断基準

- CLI、短命 worker、serverless、コンテナ起動時間が重要なサービスでは候補になる。
- plugin、動的 script、広範な reflection、実行時型探索が中心のアプリでは採用コストが高い。
- まず `PublishTrimmed=true` や `PublishAot=true` を CI の検証ジョブで試し、警告をゼロに近づける。

```xml
<PropertyGroup>
  <PublishTrimmed>true</PublishTrimmed>
  <TrimMode>partial</TrimMode>
  <IsAotCompatible>true</IsAotCompatible>
</PropertyGroup>
```

## reflection と source generator

reflection で private member や未参照型を探す設計は、trimming と相性が悪くなります。JSON は source generation、DI は明示登録、mapping は source generator や手書き mapping を検討します。

```csharp
[JsonSerializable(typeof(OrderDto))]
internal partial class AppJsonContext : JsonSerializerContext;

var json = JsonSerializer.Serialize(order, AppJsonContext.Default.OrderDto);
```

どうしても reflection が必要な場合は、`DynamicallyAccessedMembers`、`DynamicDependency`、`RequiresUnreferencedCode` で制約とリスクを呼び出し元に伝えます。

## native interop との境界

NativeAOT では P/Invoke、文字列 marshaling、callback、ライブラリ解決の差が表面化しやすくなります。`LibraryImport` を優先し、対象 OS / architecture ごとの native library 配布とロード確認を CI で行います。

## レビュー観点

- trimming/AOT 警告を抑制だけで隠していないか。
- JSON、reflection、DI、plugin、native interop の動的利用が明示的に検証されているか。
- `RequiresUnreferencedCode` が public API に漏れる場合、利用者へ代替手段を示しているか。
- publish 後の成果物で smoke test を実行しているか。
