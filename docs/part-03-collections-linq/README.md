# Part 3: コレクションと LINQ

C# のコレクションと LINQ は、日常的な業務コードの読みやすさを大きく左右する。
ここでは「どの型で受け渡すか」「いつ評価されるか」「読みやすさと性能の境界をどこに置くか」を判断できるように整理する。

この Part では、コレクション型を「実装の都合」ではなく「契約」として扱う。順序、重複、検索キー、列挙回数、評価タイミングを明示できると、レビューで性能問題や副作用を見つけやすくなる。

## 読み方

- まずコレクション型の選択で、順序・重複・検索の前提を確認する。
- API 境界では `IEnumerable<T>`、`IReadOnlyCollection<T>`、`IReadOnlyList<T>` のどれを約束するかを見る。
- LINQ は「何をするか」だけでなく「いつ評価されるか」「何回列挙されるか」を確認する。
- 性能の話は、推測ではなく件数、呼び出し頻度、DB 変換、計測結果と結びつける。

## 項目一覧

- [array、List、Dictionary、HashSet](array-list-dictionary-hashset.md)
- [IEnumerable / ICollection / IReadOnlyCollection の使い分け](ienumerable-icollection-ireadonlycollection-の使い分け.md)
- [LINQ の基本、遅延評価、即時評価](linq-の基本-遅延評価-即時評価.md)
- [LINQ を使いこなす](linq-を使いこなす.md)
- [LINQ で避けたい書き方とパフォーマンス](linq-で避けたい書き方とパフォーマンス.md)

---

[章別ノート一覧に戻る](../README.md)

