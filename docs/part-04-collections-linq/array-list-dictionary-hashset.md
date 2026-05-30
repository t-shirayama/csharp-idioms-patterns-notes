# array、List、Dictionary、HashSet

## 概要

`array`、`List<T>`、`Dictionary<TKey, TValue>`、`HashSet<T>` は、用途が似て見えても得意な操作が違う。要素数が固定なら配列、順序付きで増減するなら `List<T>`、キー検索なら `Dictionary<TKey, TValue>`、重複排除や存在判定なら `HashSet<T>` を基本線にする。

型の選択は、単なる性能差ではなく「この値の意味」を読み手に伝える設計でもある。順序が契約なのか、重複を許すのか、検索キーが一意なのかを先に決めると、後続の LINQ やループも自然に短くなる。

## 使いどころ

- `array` は外部 API や固定長の値を扱うときに向く。
- `List<T>` は画面表示、集計前の一時リスト、順序が意味を持つ処理で使いやすい。
- `Dictionary<TKey, TValue>` は ID やコードから値を高速に引く場面に向く。
- `HashSet<T>` は「含まれるか」「すでに処理済みか」を判定する場面で有効。

| 型 | 向いていること | 注意点 |
| --- | --- | --- |
| `T[]` | 固定長、低レベル API との受け渡し | サイズ変更できない。公開すると要素を書き換えられる |
| `List<T>` | 順序付きの追加、画面表示用の並び | 存在判定を大量に行う用途には弱い |
| `Dictionary<TKey, TValue>` | キーから値を引く | キー重複、キーの等価性、キー変更に注意する |
| `HashSet<T>` | 重複排除、存在判定、集合演算 | 順序を契約にしない。独自型は等価性を明示する |

```csharp
var usersById = users.ToDictionary(user => user.Id);

if (usersById.TryGetValue(request.UserId, out var user))
{
    return user.DisplayName;
}
```

`ToDictionary` は重複キーがあると例外になる。重複が業務上あり得るなら、先に `GroupBy` で扱いを明示する。

```csharp
var usersByEmail = users
    .GroupBy(user => user.Email, StringComparer.OrdinalIgnoreCase)
    .ToDictionary(group => group.Key, group => group.ToList());
```

`HashSet<T>` は「重複を消す」だけでなく、「処理済みか」を明確にするためにも使える。

```csharp
var processedIds = new HashSet<string>(StringComparer.OrdinalIgnoreCase);

foreach (var id in requestedIds)
{
    if (!processedIds.Add(id))
    {
        continue;
    }

    Process(id);
}
```

## 判断基準

- 順序が意味を持つなら `List<T>` や `IReadOnlyList<T>` を候補にする。
- 一意キーで引くなら `Dictionary<TKey, TValue>` を先に考える。
- 存在判定を何度も行うなら `HashSet<T>` に寄せる。
- 件数が非常に少なく、処理が一度きりなら、読みやすい `List<T>` のままで十分なことも多い。
- 外部に公開する戻り値では、内部の `List<T>` をそのまま変更可能な形で返さない。

## 避けたい書き方

`List<T>` に対して毎回 `Any(x => x.Id == id)` を繰り返すと、件数が増えたときに線形探索が積み上がる。検索が主目的なら、最初から `Dictionary` や `HashSet` に変換するほうが意図も性能も安定する。

```csharp
// 避けたい例: ループのたびに全件探索する
foreach (var id in requestedIds)
{
    if (users.Any(user => user.Id == id))
    {
        // ...
    }
}
```

```csharp
// 良い例: 検索用の構造を先に作る
var userIds = users.Select(user => user.Id).ToHashSet();

foreach (var id in requestedIds)
{
    if (userIds.Contains(id))
    {
        // ...
    }
}
```

`Dictionary` のキーにする値は、あとから変わらない値を選ぶ。可変オブジェクトをキーにしたり、キーに関係するプロパティを書き換えたりすると、検索できなくなる原因になる。

```csharp
// 避けたい例: 可変の表示名をキーにしている
var usersByName = users.ToDictionary(user => user.DisplayName);
```

## テストしやすさ

コレクション選択はテストデータの作りやすさにも影響する。`Dictionary` を受け取る API は「キー検索が前提」と分かりやすい一方、呼び出し側の準備が少し重くなる。単に列挙できればよい引数にまで `List<T>` や `Dictionary` を要求すると、テストが余計に具体実装へ縛られる。

## レビュー観点

- 要素の順序、重複、検索キーの有無に合った型を選んでいるか。
- `Dictionary` のキーに null や重複が混ざる可能性を考慮しているか。
- `HashSet<T>` で独自型を使う場合、等価性比較の意図が明確か。
- Queue や Stack を使うべき処理を、無理に `List<T>` の先頭挿入や削除で表現していないか。
- `StringComparer.OrdinalIgnoreCase` など、文字列キーの比較ルールを明示しているか。
- 公開 API で配列や `List<T>` を返す場合、呼び出し側に変更されても問題ないか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: ConcurrentDictionary

> 対応元: P1 / `ConcurrentDictionary`、immutable collection、`FrozenDictionary` / `FrozenSet`、`Queue` / `Stack` の使いどころを補う。

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
