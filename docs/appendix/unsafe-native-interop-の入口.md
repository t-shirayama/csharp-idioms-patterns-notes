# unsafe / native interop の入口

## 概要

`unsafe` と native interop は、ポインタや OS / ネイティブライブラリとの境界を扱うための入口です。
性能や既存資産の利用に必要な場面がありますが、型安全性、メモリ安全性、プラットフォーム差を自分で引き受けることになります。

この領域では「動いた」だけでは不十分です。
所有権、解放タイミング、文字列エンコーディング、構造体レイアウト、呼び出し規約、32/64 bit 差を明示し、通常の C# コードから見える範囲を小さくします。

```csharp
internal static partial class NativeMethods
{
    [LibraryImport("kernel32.dll", SetLastError = true)]
    [return: MarshalAs(UnmanagedType.Bool)]
    internal static partial bool Beep(int frequency, int duration);
}
```

## 判断基準

- 標準ライブラリや信頼できる NuGet で足りない理由があるか。
- 性能目的なら、`Span<T>`、`MemoryMarshal`、pooling など managed な選択肢を先に測定したか。
- native resource の所有者、解放 API、例外時の後始末を説明できるか。
- 対応 OS、CPU architecture、文字コード、呼び出し規約が仕様として決まっているか。
- P/Invoke を公開 API に出さず、managed な facade を用意できるか。

## 使いどころ

- OS API や既存の C / C++ ライブラリを呼び出す。
- 画像処理、暗号、圧縮などで、測定済みの hot path を最適化する。
- NativeAOT や単一バイナリ配布で、実行環境の制約に合わせる。
- device、driver、既存社内ライブラリなど managed API が存在しない境界を扱う。

## よい例

native handle は `SafeHandle` に閉じ込め、呼び出し側に `IntPtr` を渡さないようにします。

```csharp
public sealed class NativeBufferHandle : SafeHandle
{
    public NativeBufferHandle() : base(IntPtr.Zero, ownsHandle: true)
    {
    }

    public override bool IsInvalid => handle == IntPtr.Zero;

    protected override bool ReleaseHandle()
    {
        NativeMethods.FreeBuffer(handle);
        return true;
    }
}
```

`unsafe` が必要な処理も、境界を小さくします。

```csharp
static unsafe int ReadFirstInt(ReadOnlySpan<byte> bytes)
{
    if (bytes.Length < sizeof(int))
    {
        throw new ArgumentException("At least 4 bytes are required.", nameof(bytes));
    }

    fixed (byte* pointer = bytes)
    {
        return *(int*)pointer;
    }
}
```

## 避けたい書き方

- 測定せずに `unsafe` を使い、通常の C# で十分な処理を複雑にする。
- `IntPtr` の所有権、解放責任、文字列エンコーディングを曖昧にする。
- プラットフォーム固有 API を業務ロジックから直接呼ぶ。
- `SetLastError` や `Marshal.GetLastPInvokeError()` の扱いを決めず、失敗原因を捨てる。
- struct layout を native 側と照合せず、環境によってだけ壊れる状態にする。

```csharp
// 避けたい例: 解放責任が見えず、例外時の後始末も漏れやすい。
IntPtr buffer = NativeAllocate(size);
UseBuffer(buffer);
```

```csharp
// 避けたい例: OS 固有 API が業務処理に漏れている。
if (!NativeMethods.Beep(800, 100))
{
    throw new InvalidOperationException("Beep failed.");
}
```

## テストしやすくする

native 境界は薄く包み、通常の単体テストでは managed facade を差し替えます。
結合テストでは、対象 OS と architecture を明示し、native library が見つからない場合の失敗も確認します。

```csharp
public interface ISystemSound
{
    void PlayNotification();
}

public sealed class WindowsSystemSound : ISystemSound
{
    public void PlayNotification()
    {
        if (!NativeMethods.Beep(800, 100))
        {
            throw new Win32Exception(Marshal.GetLastPInvokeError());
        }
    }
}
```

## レビュー観点

- `unsafe` や P/Invoke が小さな境界に閉じ、通常コードから直接見えないか。
- `SafeHandle`、`IDisposable`、`try` / `finally` などで native resource の寿命を管理しているか。
- x86 / x64、Windows / Linux、文字コード、構造体レイアウトの差を確認しているか。
- 代替として標準ライブラリや既存 NuGet で足りない理由が説明できるか。
- `LibraryImport` / `DllImport` の marshaling 指定、`SetLastError`、文字列エンコーディングが native 定義と合っているか。
- CI で対象 platform の少なくとも代表ケースを動かせるか。動かせない場合、手動確認手順が残っているか。

---

[Appendix README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: LibraryImport と DllImport の使い分け

> 対応元: P0 / `LibraryImport` と `DllImport` の使い分け、文字列 marshaling、`SetLastError`、`NativeLibrary.SetDllImportResolver`、pinning / `GCHandle`、endianness、`MemoryMarshal` の注意を拡充する。

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
