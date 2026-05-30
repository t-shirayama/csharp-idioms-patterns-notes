# enum、flags enum、代替設計

## 概要

enum は、閉じた選択肢を軽量に表すための型です。flags enum は複数選択のビット集合に向きます。一方で、表示名、振る舞い、外部設定、ライフサイクルを持つ値は、class や record の方が扱いやすい場合があります。

## 判断基準

enum は「コード上で選択肢が閉じている」「値ごとのデータや振る舞いが少ない」「追加頻度が低い」場合に向きます。画面表示名、並び順、利用可否、外部連携コード、権限判定などが値ごとに増えてきたら、enum だけで表すより専用型や設定テーブルに寄せる方が変更に強くなります。

永続化する enum は、将来の変更を前提に扱います。数値保存は省スペースですが、並び替えや途中挿入に弱くなります。文字列保存は読みやすい一方、リネームの移行が必要になります。

## 使いどころ

- 状態や種別がコード上で閉じており、頻繁に増減しない場合に enum を使う。
- 権限やオプションの組み合わせには `[Flags]` を使う。
- DB に保存する場合は、数値か文字列かを変更耐性込みで決める。

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

## 避けたい書き方

- enum の `0` に意味を持たせず、初期値が不正状態になる。
- flags enum で `1, 2, 3` のようにビット値ではない連番を使う。
- 画面表示名や業務ロジックを enum 名の文字列変換に依存する。

```csharp
public enum Permission
{
    Read = 1,
    Write = 2,
    Approve = 3
}
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

## レビュー観点

- `None = 0` や `Unknown = 0` など、既定値の意味が定義されているか。
- enum を switch する箇所で、新しい値を追加したときにコンパイルやテストで気付けるか。
- 永続化される enum の値変更が既存データに影響しないか。
- 値ごとに振る舞いが増えているなら、record、class、Strategy などへの移行を検討したか。
- flags enum で組み合わせ値と単独値が混ざっても判定が破綻しないか。
- 外部 API や DB の値を enum に変換する境界で、不明値の扱いを決めているか。

---

[Part README に戻る](README.md)


