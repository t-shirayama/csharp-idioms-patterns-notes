# enum、flags enum、代替設計

## 概要

enum は、閉じた選択肢を軽量に表すための型です。flags enum は複数選択のビット集合に向きます。一方で、表示名、振る舞い、外部設定、ライフサイクルを持つ値は、class や record の方が扱いやすい場合があります。

実務では、enum は「分岐を減らす道具」ではなく「選択肢を型として閉じる道具」として考えます。値が増えたときにどこを直すべきか分かるなら enum は有効です。値ごとのルールがあちこちに散るなら、別の設計を検討する合図です。

## 判断基準

enum は「コード上で選択肢が閉じている」「値ごとのデータや振る舞いが少ない」「追加頻度が低い」場合に向きます。画面表示名、並び順、利用可否、外部連携コード、権限判定などが値ごとに増えてきたら、enum だけで表すより専用型や設定テーブルに寄せる方が変更に強くなります。

永続化する enum は、将来の変更を前提に扱います。数値保存は省スペースですが、並び替えや途中挿入に弱くなります。文字列保存は読みやすい一方、リネームの移行が必要になります。

`0` の扱いは最初に決めます。C# の enum は default が 0 なので、`None`、`Unknown`、`Unspecified` のように意味を置くか、0 を不正値として入口で弾く必要があります。何も定義しないまま 0 が流れると、初期化漏れと正しい値を区別できません。

flags enum は、独立したオンオフの組み合わせに向きます。状態遷移や排他的な分類には向きません。`Submitted | Canceled` のような組み合わせが業務上あり得ないなら、flags ではなく通常の enum や状態オブジェクトで表します。

## 使いどころ

- 状態や種別がコード上で閉じており、頻繁に増減しない場合に enum を使う。
- 権限やオプションの組み合わせには `[Flags]` を使う。
- DB に保存する場合は、数値か文字列かを変更耐性込みで決める。
- 外部値を enum に変換する境界では、不明値を拒否するか `Unknown` に寄せる。
- 値ごとの表示や外部コードは、変換処理を一箇所に集める。

```csharp
[Flags]
public enum Permission
{
    None = 0,
    Read = 1,
    Write = 2,
    Approve = 4
}

var canEdit = (permissions & Permission.Write) == Permission.Write;
```

外部 API のコードは、enum へ直接キャストせず境界で変換します。

```csharp
public static OrderStatus ParseStatus(string code) => code switch
{
    "draft" => OrderStatus.Draft,
    "submitted" => OrderStatus.Submitted,
    "canceled" => OrderStatus.Canceled,
    _ => OrderStatus.Unknown
};
```

## 良い例

flags enum では `None = 0` を置き、各値は 2 のべき乗にします。よく使う組み合わせは名前を付けると、呼び出し側の意図が読みやすくなります。

```csharp
[Flags]
public enum Permission
{
    None = 0,
    Read = 1 << 0,
    Write = 1 << 1,
    Approve = 1 << 2,
    Manage = Read | Write | Approve
}

var canManage = permissions.HasFlag(Permission.Manage);
```

値ごとの表示名は、`ToString()` や enum 名に依存させず、変換の責務としてまとめます。未対応値を例外にすると、値追加時に気付きやすくなります。

```csharp
public static string ToDisplayName(this OrderStatus status) => status switch
{
    OrderStatus.Unknown => "不明",
    OrderStatus.Draft => "下書き",
    OrderStatus.Submitted => "申請済み",
    OrderStatus.Canceled => "取消",
    _ => throw new ArgumentOutOfRangeException(nameof(status), status, null)
};
```

## 避けたい書き方

- enum の `0` に意味を持たせず、初期値が不正状態になる。
- flags enum で `1, 2, 3` のようにビット値ではない連番を使う。
- 画面表示名や業務ロジックを enum 名の文字列変換に依存する。
- 通常の enum で表すべき排他的な状態を flags enum にする。
- 外部入力の数値を enum へキャストし、不明値を正規化しない。

```csharp
public enum Permission
{
    Read = 1,
    Write = 2,
    Approve = 3
}
```

```csharp
// 避けたい例: DB の不正値が enum として内部へ入る
var status = (OrderStatus)record.Status;
```

switch の `default` で何でも既定値に落とすと、値が増えたときに気付きにくくなります。業務上あり得ない値は例外にし、外部入力としてあり得る不明値は `Unknown` として明示します。

```csharp
// 避けたい例: 新しい状態が追加されても通常扱いになってしまう
var label = status switch
{
    OrderStatus.Canceled => "取消",
    _ => "通常"
};
```

## 改善例

値ごとに表示名や判定が増える場合は、変換処理を一箇所に寄せます。さらに値ごとの振る舞いが増えるなら、Strategy や record に移すことを検討します。

```csharp
public static string ToDisplayName(this OrderStatus status) => status switch
{
    OrderStatus.Draft => "下書き",
    OrderStatus.Submitted => "申請済み",
    OrderStatus.Canceled => "取消",
    _ => throw new ArgumentOutOfRangeException(nameof(status), status, null)
};
```

値ごとの振る舞いが主役になったら、enum を分岐のキーにし続けるより、振る舞いを持つ型へ移します。すぐに大きな設計へ寄せず、まずは小さな record や設定テーブルでも十分なことがあります。

```csharp
public sealed record ShippingMethod(
    string Code,
    string DisplayName,
    decimal BaseFee);

public static readonly ShippingMethod Express = new("express", "速達", 800m);
```

## レビュー観点

- `None = 0` や `Unknown = 0` など、既定値の意味が定義されているか。
- enum を switch する箇所で、新しい値を追加したときにコンパイルやテストで気付けるか。
- 永続化される enum の値変更が既存データに影響しないか。
- 値ごとに振る舞いが増えているなら、record、class、Strategy などへの移行を検討したか。
- flags enum で組み合わせ値と単独値が混ざっても判定が破綻しないか。
- 外部 API や DB の値を enum に変換する境界で、不明値の扱いを決めているか。
- `HasFlag` やビット演算が、`None` や組み合わせ値で意図どおり動くか。
- enum 名を画面表示や外部契約に流用していないか。

## テスト観点

- default 値、未知の外部値、廃止済みの値をどう扱うかをテストする。
- enum 値を追加したとき、表示名、変換、権限判定のテストが失敗するか確認する。
- flags enum は、単独値、複数組み合わせ、`None`、定義外ビットを確認する。
- 永続化する enum は、既存データの数値や文字列が移行後も読めるかを押さえる。
- 代替設計へ移した場合、値ごとの振る舞いが分岐の散在ではなく型や設定に集まっているかを見る。

---

[Part README に戻る](README.md)


