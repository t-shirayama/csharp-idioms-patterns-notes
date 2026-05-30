# Part 11 コードレビューチェックリスト

C# のステップアップ機能を使うレビューでは、構文の正しさだけでなく、抽象化の妥当性、所有権、評価タイミング、テストしやすさを確認します。
各項目は、コード、テスト、設計メモ、レビューコメントのいずれかで根拠を確認できる粒度にしています。

## generics と constraints

- [ ] generic method / type は、複数の具体型で同じ意味の処理を共有しているか。単に引数型を隠すためだけに使っていないか。
- [ ] 型パラメータ名は `T`、`TKey`、`TValue`、`TEntity` など、役割が読み取れる名前になっているか。
- [ ] constraints は、実装が必要とする能力を過不足なく表しているか。
- [ ] `where T : class`、`struct`、`notnull`、`unmanaged`、`new()` の意味を nullable と default 値の観点で確認しているか。
- [ ] constraints が弱すぎて、reflection、`as`、強制キャスト、例外による分岐に逃げていないか。
- [ ] 型パラメータが増えすぎて、呼び出し側が型推論に頼らないと意味を追えないAPIになっていないか。

## variance

- [ ] `out T` は戻り値としての利用に限定され、入力位置に使われていないか。
- [ ] `in T` は引数としての利用に限定され、戻り値として返していないか。
- [ ] `IEnumerable<Derived>` を `IEnumerable<Base>` として扱うなど、共変性の利用が読み手に自然か。
- [ ] mutable collection で variance が使えない理由を理解し、無理なキャストで回避していないか。
- [ ] delegate / interface の variance は、API利用側の代入互換性を改善しているか。

## delegates と events

- [ ] delegate は、単純な処理差し替え、フィルタ、変換、コールバックとして十分に小さい責務か。
- [ ] event は、通知の所有権を発火側に残す目的で公開されているか。
- [ ] event の購読解除が必要なライフサイクルで、解除漏れを防ぐ設計になっているか。
- [ ] event handler 内の例外が、発火側の処理全体を壊してよいか検討されているか。
- [ ] `Action` / `Func` で意図が分かりにくい場合、名前付き delegate や interface の方が適切でないか。
- [ ] テストで callback が呼ばれた回数、引数、順序、例外時の振る舞いを確認できるか。

## extension methods

- [ ] extension method は、対象型の自然な補助操作として読めるか。
- [ ] 業務上重要な状態変更や外部I/Oを、extension method に隠していないか。
- [ ] namespace import によって、呼び出し可能なメソッドが予想外に増えないか。
- [ ] 既存インスタンスメソッドと紛らわしい名前、過度に一般的な名前を付けていないか。
- [ ] null を受け取る可能性がある extension method では、ガードと nullable 注釈が明確か。
- [ ] テストでは、正常系だけでなく null、空コレクション、境界値を確認しているか。

## attributes

- [ ] attribute は、宣言的に表す価値があるメタデータか。通常のコードで十分な処理を隠していないか。
- [ ] attribute を読む側の仕組みが明確で、起動時やテストで検証できるか。
- [ ] reflection を使う場合、性能、AOT、trimming、アクセス権限への影響を考慮しているか。
- [ ] validation、serialization、DI登録などの attribute が、実行時の契約とずれていないか。
- [ ] attribute の引数は compile-time constant に限定されるため、設定値や環境依存値を無理に持たせていないか。
- [ ] source generator や analyzer と組み合わせる場合、生成コードの確認方法があるか。

## iterators と yield

- [ ] `IEnumerable<T>` を返すメソッドが遅延評価であることは、呼び出し側にとって自然か。
- [ ] 例外がメソッド呼び出し時ではなく列挙時に発生することを、テストで確認しているか。
- [ ] `yield` 内で扱う `IDisposable` は、列挙終了時や途中終了時にも解放されるか。
- [ ] sequence を複数回列挙した場合に、重い計算、I/O、副作用が繰り返されないか。
- [ ] 即時評価すべき箇所では `ToList()` や `ToArray()` で境界を明確にしているか。
- [ ] async stream を使う場合、`CancellationToken` と `await foreach` のキャンセルが考慮されているか。

## modern syntax

- [ ] primary constructor は、依存関係や初期化値の意図を読みやすくしているか。
- [ ] コンストラクタ本体で validation や変換が多い場合、従来のコンストラクタの方が読みやすくないか。
- [ ] collection expression、target-typed `new`、file-local type などの新構文は、対象 C# バージョンで利用可能か。
- [ ] 新構文が混在して、同じ意味の書き方がファイルごとにばらついていないか。
- [ ] 短く書くことで、null 契約、所有権、例外、validation の意図が見えにくくなっていないか。

## テスト観点

- [ ] generic API は、代表的な複数の型で同じ契約を満たすことを確認しているか。
- [ ] callback / event は、未購読、複数購読、解除後、handler 例外のケースを確認しているか。
- [ ] attribute ベースの処理は、attribute なし、誤った設定、複数指定のケースを確認しているか。
- [ ] iterator は、途中で列挙をやめた場合、複数回列挙した場合、例外が出る場合を確認しているか。
- [ ] 新構文への置き換え後も、公開API、serialization、DI、テストデータ生成への影響がないか確認しているか。

---

[Part README に戻る](README.md)
