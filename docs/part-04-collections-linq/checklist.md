# Part 4 コードレビュー用チェックリスト

コレクションと LINQ は、型の選び方、列挙回数、評価タイミングによって読みやすさと性能が大きく変わります。
レビューでは「動くか」だけでなく、データ量、順序、重複、検索キー、API 境界の契約がコードから読み取れるかを確認します。

## コレクション型の選択

- [ ] 順序が意味を持つ処理では、`List<T>` や `IReadOnlyList<T>` など、順序を表す型になっている。
- [ ] 順序が意味を持たない集合処理で、偶然の `Dictionary` / `HashSet` の列挙順に依存していない。
- [ ] 重複を許すかどうかが仕様として分かり、重複を避けるなら `HashSet<T>` やキー制約を検討している。
- [ ] ID やコードで何度も検索する処理では、毎回 `FirstOrDefault` せず `Dictionary<TKey, TValue>` による索引化を検討している。
- [ ] 存在判定を繰り返す処理では、`List<T>.Contains` の線形探索ではなく `HashSet<T>` が適切か確認している。
- [ ] 1 つのキーに複数要素がぶら下がる仕様を、`Dictionary<TKey, TValue>` に押し込まず `Lookup` や `Dictionary<TKey, List<T>>` として表している。
- [ ] 先頭への追加や削除を多用する処理で、`List<T>` ではなく `Queue<T>`、`Stack<T>`、`LinkedList<T>` などのほうが自然ではないか確認している。
- [ ] `Dictionary` や `HashSet` のキーが文字列の場合、`StringComparer.Ordinal` / `OrdinalIgnoreCase` など比較ルールを明示している。
- [ ] 独自型を `Dictionary` のキーや `HashSet` の要素にする場合、等価性比較の意図が `Equals`、`GetHashCode`、`IEqualityComparer<T>` で表現されている。
- [ ] mutable なプロパティを等価性や hash code に使う型を、`HashSet` の要素や `Dictionary` のキーにしていない。
- [ ] キー検索では `ContainsKey` の後に indexer で取り直すのではなく、値が必要なら `TryGetValue` を使っている。
- [ ] 公開 API で内部の `List<T>` や配列をそのまま返し、呼び出し側から変更できる状態にしていない。

## API 境界の契約

- [ ] 引数が `IEnumerable<T>` の場合、メソッド内で 1 回だけ列挙すれば足りる。
- [ ] 件数を使う処理では、`IEnumerable<T>` ではなく `IReadOnlyCollection<T>` など `Count` を契約に含める型を検討している。
- [ ] インデックスアクセスや順序が仕様なら、`IReadOnlyCollection<T>` ではなく `IReadOnlyList<T>` を使う理由がある。
- [ ] 呼び出し先で追加や削除を行う責務がないのに、引数を `ICollection<T>` や `List<T>` にしていない。
- [ ] 呼び出し先で null を許すか空コレクションを期待するかが明確で、null と空を曖昧に扱っていない。
- [ ] 戻り値を `IEnumerable<T>` にする場合、遅延評価のまま返すのか、`ToList()` で固定して返すのかを説明できる。
- [ ] DB、ファイル、HTTP response など寿命のあるリソースをまたぐ遅延列挙を外へ漏らしていない。
- [ ] repository や service の境界から `IQueryable<T>` を漏らし、呼び出し側に DB クエリの組み立て責務を移していない。
- [ ] `IReadOnlyCollection<T>` がコレクション操作を制限するだけで、要素自体の不変性までは保証しないことを前提にしている。

## LINQ の読みやすさ

- [ ] LINQ チェーンが長くなりすぎた場合、途中結果に業務用語の変数名を付けて意図を残している。
- [ ] `Select`、`Where`、`GroupBy` の中にログ出力、状態変更、外部 I/O などの副作用を混ぜていない。
- [ ] 変更を伴う処理は、LINQ で無理に書かず `foreach` に分けている。
- [ ] `First`、`FirstOrDefault`、`Single`、`SingleOrDefault` の選択が、データ不整合を許すか検知するかという意図に合っている。
- [ ] `FirstOrDefault` や `SingleOrDefault` の戻り値が null になり得る場合、その後の処理で扱っている。
- [ ] `Any` は存在確認、`Count` は件数そのものが必要なとき、という使い分けになっている。
- [ ] 複数条件の並び替えで、2 回目以降に `OrderBy` ではなく `ThenBy` / `ThenByDescending` を使っている。
- [ ] `Aggregate` を使う前に、`Sum`、`Any`、`All`、`MaxBy`、`MinBy` など専用演算子で表せないか確認している。
- [ ] `DistinctBy` や `GroupBy` で「どの要素が代表として残るか」が、並び順を含めて明確になっている。
- [ ] `SelectMany`、`Join`、`GroupJoin` を使う処理で、内側と外側の対応関係、未一致データの扱いが読み取れる。
- [ ] `DefaultIfEmpty`、`FirstOrDefault`、`SingleOrDefault` の default 値が有効な値と衝突しないか確認している。

## 遅延評価と materialize

- [ ] query を変数に置いたとき、列挙されるタイミングがレビューで追える。
- [ ] 同じ `IEnumerable<T>` を複数回列挙する場合、再実行のコストや副作用が問題にならない。
- [ ] LINQ の条件式が外側の mutable 変数を参照している場合、列挙時点の値で評価されることが意図通りである。
- [ ] 複数回使う、ログに出す、DB クエリをそこで実行したいなど、`ToList()` / `ToArray()` の理由が明確である。
- [ ] `Where(...).ToList().Select(...)` のように、途中で不要な中間コレクションを作っていない。
- [ ] `OrderBy`、`GroupBy`、`ToList` など全体を保持しやすい操作が、大量データに対して無自覚に使われていない。
- [ ] `AsEnumerable` によって DB 側で絞れる処理をメモリ上へ移していない。
- [ ] EF Core などの `IQueryable<T>` では、C# の式が期待通り SQL に変換されるか確認している。
- [ ] `Include` 後の LINQ や navigation property の列挙で、N+1 や意図しない追加クエリが起きないか確認している。

## パフォーマンスとデータ量

- [ ] 内側ループで `Any`、`FirstOrDefault`、`SingleOrDefault` を繰り返し、実質的に O(n * m) になっていない。
- [ ] nested loop を避けるために `ToDictionary`、`ToLookup`、`HashSet` を使う場合、キー重複や comparer の仕様も同時に確認している。
- [ ] 性能改善のための `Dictionary` 化、`HashSet` 化、`ToList` 追加には、件数、呼び出し頻度、計測結果のいずれかの根拠がある。
- [ ] 少件数で一度だけ処理するコードを、読みづらい最適化で複雑にしていない。
- [ ] `Count()` が安い `ICollection<T>` なのか、全列挙になり得る `IEnumerable<T>` なのかを意識している。
- [ ] `ToDictionary` でキー重複が起きた場合の例外が、仕様上問題ないか確認している。
- [ ] 大量データを扱う処理で、ソート、グループ化、distinct、join のメモリ使用量を見積もっている。
- [ ] 最大値や最小値だけが必要な処理で、全体を `OrderBy(...).First()` / `Last()` していない。

## テスト観点

- [ ] 空コレクション、1 件、複数件、重複あり、キー欠落ありのケースを確認している。
- [ ] 順序が意味を持つ処理では、期待する並び順をテストで固定している。
- [ ] 大小文字違い、全角半角、前後空白など、文字列キーの comparer に関わるケースを確認している。
- [ ] `Single` で不整合を検知する処理では、0 件と複数件の失敗ケースをテストしている。
- [ ] 遅延評価を含む処理では、複数回列挙や列挙前後の元データ変更で振る舞いが崩れないか確認している。
- [ ] DB LINQ では、必要に応じて発行 SQL、N+1、メモリ上評価への切り替わりを確認している。

---

[Part 4 README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: チェックリストで先行している論点を本文へのリンクや「詳しくは各章へ」で接続する。

> 対応元: P2 / チェックリストで先行している論点を本文へのリンクや「詳しくは各章へ」で接続する。

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
