# LINQ の過剰使用

## 概要

LINQ は意図を宣言的に書ける強力な道具です。一方で、遅延実行、多重列挙、中間コレクション、ラムダによる allocation など、コストが見えにくいことがあります。

読みやすさが上がるなら LINQ は有効です。ただし、同じ sequence を何度も列挙する処理や、大量件数の hot path では `for` / `foreach` に戻す判断も必要です。

```csharp
// 避けたい例: source が DB や高コストな列挙の場合、2 回評価される。
if (orders.Any(order => order.IsOverdue))
{
    var total = orders.Sum(order => order.Amount);
    Console.WriteLine(total);
}
```

```csharp
var hasOverdue = false;
var total = 0m;

foreach (var order in orders)
{
    hasOverdue |= order.IsOverdue;
    total += order.Amount;
}

if (hasOverdue)
{
    Console.WriteLine(total);
}
```

## 判断基準

- `IEnumerable<T>` は再列挙できるとは限らない。DB、ファイル、外部 API、yield の可能性を見る。
- `IQueryable<T>` は「C# の処理」ではなく「変換されるクエリ」として読む。
- `ToList()` は境界を固定する意図があるときに使い、惰性で挟まない。
- hot path では、LINQ の読みやすさと allocation、delegate 呼び出し、多重列挙のコストを比べる。
- 件数が小さい設定値や画面用整形では、LINQ の可読性を優先してよいことが多い。

## 使いどころ

- フィルタ、射影、集計などの意図を短く表したい通常のアプリケーションコード。
- 件数が小さく、可読性の利益が実行時コストを上回る場面。
- LINQ to Objects と DB LINQ の違いを理解したうえで、クエリとして表現したい場面。

## 避けたい書き方

- `Where(...).ToList().Select(...).ToList()` のように中間コレクションを重ねる。
- `IEnumerable<T>` を何度も列挙して、DB や外部 API 呼び出しが繰り返される。
- `IQueryable<T>` に、DB へ変換できない .NET メソッドや副作用を混ぜる。
- LINQ の途中でログ出力や状態変更を行い、遅延実行でタイミングが読めなくなる。

```csharp
// 避けたい例: IQueryable にローカルメソッドを混ぜると変換できない場合がある。
var users = db.Users.Where(user => IsTarget(user.Email));
```

```csharp
var users = db.Users
    .Where(user => user.Email.EndsWith("@example.com"));
```

## レビュー観点

- `IEnumerable<T>`、`IReadOnlyList<T>`、`IQueryable<T>` のどれを扱っているか明確か。
- `Count()`、`Any()`、`Sum()` などが同じ sequence に対して重複していないか。
- `ToList()` は境界を固定する意図があるか、それとも惰性で入っているだけか。
- 可読性が下がるほどループ化していないか。必要ならコメントではなく処理名で意図を表す。
- `IQueryable<T>` の結果 SQL や実行タイミングを確認しているか。
- 遅延実行によって例外や副作用の発生タイミングが呼び出し側に漏れていないか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: List<T>.Contains と HashSet<T>

> 対応元: P1 / `List<T>.Contains` と `HashSet<T>`、`OrderBy` / `GroupBy` / `Distinct`、`AsEnumerable()` 後の DB 外処理を追加する。

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
