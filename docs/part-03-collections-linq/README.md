# Part 3: コレクションと LINQ

C# のコレクションと LINQ は、業務コードの読みやすさ、性能、境界設計に直結します。
この Part では、`List<T>` を便利な入れ物として使うだけでなく、順序、重複、検索キー、変更可否、評価タイミングをコード上の契約として表す判断力を扱います。

LINQ は宣言的に書ける一方で、遅延評価、複数回列挙、DB クエリへの変換、不要な allocation などの落とし穴もあります。
レビューでは「短く書けているか」だけでなく、「どこで評価されるか」「件数が増えても同じ判断でよいか」「API 境界で何を約束しているか」を確認します。

## このPartで身につくこと

- `array`、`List<T>`、`Dictionary<TKey,TValue>`、`HashSet<T>` を目的で選び分ける。
- `IEnumerable<T>`、`ICollection<T>`、`IReadOnlyCollection<T>`、`IReadOnlyList<T>` を API 境界の契約として使う。
- LINQ の遅延評価と即時評価を区別し、副作用や複数回列挙を避ける。
- `Select`、`Where`、`GroupBy`、`Join`、`Aggregate` を、読みやすさと実行コストの両面から選ぶ。
- 大量データ、DB クエリ、外部 I/O を含む処理で、LINQ を使う範囲と materialize する位置を判断する。

## 読む順番

1. まず [array、List、Dictionary、HashSet](array-list-dictionary-hashset.md) で、型ごとの前提を整理する。
2. 次に [IEnumerable / ICollection / IReadOnlyCollection / IReadOnlyList の使い分け](ienumerable-icollection-ireadonlycollection-の使い分け.md) で、メソッド引数と戻り値の設計を見る。
3. [LINQ の基本、遅延評価、即時評価](linq-の基本-遅延評価-即時評価.md) で、評価タイミングと列挙回数を押さえる。
4. [LINQ を使いこなす](linq-を使いこなす.md) で、集計、結合、グループ化を実務コードに落とし込む。
5. 最後に [LINQ で避けたい書き方とパフォーマンス](linq-で避けたい書き方とパフォーマンス.md) と [コードレビュー用チェックリスト](checklist.md) で、レビュー観点に変換する。

## 重要トピック

- コレクション型は、順序、重複、検索の頻度、変更可否を表す設計情報である。
- `IEnumerable<T>` は「列挙できる」だけを約束し、件数取得や複数回列挙の安さは約束しない。
- `ToList()` や `ToArray()` は悪ではないが、評価位置、再利用、DB クエリ発行回数を意識して置く。
- LINQ は変換パイプラインを読みやすくする道具であり、複雑な業務分岐を無理に一文へ押し込む道具ではない。
- `Dictionary` と `HashSet` は検索を明確に速くする一方、キーの一意性、比較規則、順序への期待をレビューする必要がある。

## 実務での使いどころ

- API、サービス、ユースケース層の引数と戻り値で、呼び出し側に必要以上の変更権限を渡さない。
- CSV、JSON、DB 結果などの外部データを、早い段階で意味のあるコレクションへ整形する。
- 集計、フィルタ、射影、グループ化を LINQ で表現し、業務ルールの流れを追いやすくする。
- N+1、巨大な中間リスト、複数回列挙が問題になる箇所では、materialize、辞書化、ループへの展開を検討する。
- テストでは、空、1件、複数件、重複キー、null 要素、順序差分を固定して仕様を明確にする。

## よくある落とし穴

- `IEnumerable<T>` を何度も列挙し、DB クエリや外部 I/O が複数回走る。
- `First()`、`Single()`、`ToDictionary()` を、該当なしや重複の扱いを決めないまま使う。
- LINQ の途中に副作用を混ぜ、評価タイミングが変わると挙動も変わる。
- `GroupBy` や `Join` を使えば読みやすい場面で、ネストしたループと条件分岐を増やす。
- 逆に、単純なループの方が明確な処理を、長い LINQ チェーンにしてデバッグしづらくする。
- DB プロバイダーが変換できない式を LINQ に混ぜ、実行時エラーや想定外のクライアント評価を招く。

## レビュー観点

- このコレクション型は、順序、重複、検索、変更可否の前提と合っているか。
- API 境界で `List<T>` を要求している理由はあるか。読み取りだけなら `IReadOnlyCollection<T>` などで足りないか。
- LINQ の評価タイミングは明確か。`ToList()` の位置は意図的か。
- 空、該当なし、複数該当、重複キーの扱いがコードとテストで表現されているか。
- 件数が増えたとき、`Contains`、ネストループ、`OrderBy`、`GroupBy` のコストが問題にならないか。
- DB や外部 I/O に対する LINQ で、発行されるクエリや列挙回数を確認しているか。

## 記事一覧

- [コードレビュー用チェックリスト](checklist.md): コレクションと LINQ のレビューで見るべき契約、評価、性能、境界条件を確認する。
- [array、List、Dictionary、HashSet](array-list-dictionary-hashset.md): 代表的なコレクション型を、順序、重複、検索、変更可否の観点で選び分ける。
- [IEnumerable / ICollection / IReadOnlyCollection / IReadOnlyList の使い分け](ienumerable-icollection-ireadonlycollection-の使い分け.md): API 境界で何を約束し、何を隠すべきかを整理する。
- [LINQ の基本、遅延評価、即時評価](linq-の基本-遅延評価-即時評価.md): LINQ の評価タイミング、列挙回数、materialize の判断を押さえる。
- [LINQ を使いこなす](linq-を使いこなす.md): 射影、抽出、集計、結合、グループ化を実務コードで読みやすく使う。
- [LINQ で避けたい書き方とパフォーマンス](linq-で避けたい書き方とパフォーマンス.md): 過剰な LINQ、複数回列挙、DB 変換、allocation の落とし穴を避ける。

---

[章別ノート一覧に戻る](../index.md)
