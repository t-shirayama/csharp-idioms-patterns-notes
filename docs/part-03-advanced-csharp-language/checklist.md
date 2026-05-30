# Part 3 コードレビューチェックリスト

C# のステップアップ機能を使うレビューでは、構文の正しさだけでなく、抽象化の妥当性、所有権、評価タイミング、テストしやすさを確認します。
各項目は、コード、テスト、設計メモ、レビューコメントのいずれかで根拠を確認できる粒度にしています。

## generics と constraints

- [ ] generic method / type は、複数の具体型で同じ意味の処理を共有しているか。単に引数型を隠すためだけに使っていないか。
- [ ] 型パラメータ名は `T`、`TKey`、`TValue`、`TEntity` など、役割が読み取れる名前になっているか。
- [ ] constraints は、実装が必要とする能力を過不足なく表しているか。
- [ ] `where T : class`、`struct`、`notnull`、`unmanaged`、`new()` の意味を nullable と default 値の観点で確認しているか。
- [ ] constraints が弱すぎて、reflection、`as`、強制キャスト、例外による分岐に逃げていないか。
- [ ] 型パラメータが増えすぎて、呼び出し側が型推論に頼らないと意味を追えないAPIになっていないか。
- [ ] public API の constraint は、将来拡張のために弱めすぎず、呼び出し側に期待してよい能力を示しているか。

## variance

- [ ] `out T` は戻り値としての利用に限定され、入力位置に使われていないか。
- [ ] `in T` は引数としての利用に限定され、戻り値として返していないか。
- [ ] `IEnumerable<Derived>` を `IEnumerable<Base>` として扱うなど、共変性の利用が読み手に自然か。
- [ ] mutable collection で variance が使えない理由を理解し、無理なキャストで回避していないか。
- [ ] delegate / interface の variance は、API利用側の代入互換性を改善しているか。
- [ ] variance を使いたいが使えない場合、読み取りと書き込みの責務を同じ interface に混ぜていないか。

## delegates と events

- [ ] delegate は、単純な処理差し替え、フィルタ、変換、コールバックとして十分に小さい責務か。
- [ ] event は、通知の所有権を発火側に残す目的で公開されているか。
- [ ] event の購読解除が必要なライフサイクルで、解除漏れを防ぐ設計になっているか。
- [ ] event handler 内の例外が、発火側の処理全体を壊してよいか検討されているか。
- [ ] `Action` / `Func` で意図が分かりにくい場合、名前付き delegate や interface の方が適切でないか。
- [ ] テストで callback が呼ばれた回数、引数、順序、例外時の振る舞いを確認できるか。
- [ ] 非同期 callback / event handler は、`async void` に失敗を隠さず、例外と cancellation の扱いを説明できるか。

## extension methods

- [ ] extension method は、対象型の自然な補助操作として読めるか。
- [ ] 業務上重要な状態変更や外部I/Oを、extension method に隠していないか。
- [ ] namespace import によって、呼び出し可能なメソッドが予想外に増えないか。
- [ ] 既存インスタンスメソッドと紛らわしい名前、過度に一般的な名前を付けていないか。
- [ ] null を受け取る可能性がある extension method では、ガードと nullable 注釈が明確か。
- [ ] テストでは、正常系だけでなく null、空コレクション、境界値を確認しているか。
- [ ] `IQueryable<T>` 向け extension method は、LINQ provider が変換できる式だけで構成されているか。
- [ ] extension method の namespace は、利用範囲と業務文脈が分かる粒度になっているか。

## attributes

- [ ] attribute は、宣言的に表す価値があるメタデータか。通常のコードで十分な処理を隠していないか。
- [ ] attribute を読む側の仕組みが明確で、起動時やテストで検証できるか。
- [ ] reflection を使う場合、性能、AOT、trimming、アクセス権限への影響を考慮しているか。
- [ ] validation、serialization、DI登録などの attribute が、実行時の契約とずれていないか。
- [ ] attribute の引数は compile-time constant に限定されるため、設定値や環境依存値を無理に持たせていないか。
- [ ] source generator や analyzer と組み合わせる場合、生成コードの確認方法があるか。
- [ ] 独自 attribute では `AttributeUsage` の `AllowMultiple`、`Inherited`、対象範囲が意図どおりか。
- [ ] trimming や AOT の警告がある場合、suppression ではなく読み取り対象の設計で説明できるか。

## iterators と yield

- [ ] `IEnumerable<T>` を返すメソッドが遅延評価であることは、呼び出し側にとって自然か。
- [ ] 例外がメソッド呼び出し時ではなく列挙時に発生することを、テストで確認しているか。
- [ ] `yield` 内で扱う `IDisposable` は、列挙終了時や途中終了時にも解放されるか。
- [ ] sequence を複数回列挙した場合に、重い計算、I/O、副作用が繰り返されないか。
- [ ] 即時評価すべき箇所では `ToList()` や `ToArray()` で境界を明確にしているか。
- [ ] async stream を使う場合、`CancellationToken` と `await foreach` のキャンセルが考慮されているか。
- [ ] 引数 validation を呼び出し時に失敗させたい iterator では、外側メソッドと内側 iterator を分けているか。
- [ ] public API 名から、一度だけ消費する stream なのか、何度でも列挙できる集合なのか読み取れるか。

## primary constructors と現代構文

- [ ] primary constructor は、依存関係や初期化値の意図を読みやすくしているか。
- [ ] コンストラクタ本体で validation や変換が多い場合、従来のコンストラクタの方が読みやすくないか。
- [ ] collection expression、target-typed `new`、file-local type などの新構文は、対象 C# バージョンで利用可能か。
- [ ] 新構文が混在して、同じ意味の書き方がファイルごとにばらついていないか。
- [ ] 短く書くことで、null 契約、所有権、例外、validation の意図が見えにくくなっていないか。
- [ ] class の primary constructor と record の primary constructor の違いにより、公開 property や等価性の期待がずれていないか。
- [ ] `required` は初期化漏れ対策として使い、値の妥当性は validator や constructor guard で確認しているか。

## テスト観点

- [ ] generic API は、代表的な複数の型で同じ契約を満たすことを確認しているか。
- [ ] callback / event は、未購読、複数購読、解除後、handler 例外のケースを確認しているか。
- [ ] attribute ベースの処理は、attribute なし、誤った設定、複数指定のケースを確認しているか。
- [ ] iterator は、途中で列挙をやめた場合、複数回列挙した場合、例外が出る場合を確認しているか。
- [ ] 新構文への置き換え後も、公開API、serialization、DI、テストデータ生成への影響がないか確認しているか。
- [ ] source generator / analyzer を伴う機能は、生成結果、診断、誤用時のメッセージを確認しているか。
- [ ] `required`、nullable、serializer、configuration binding の入口ごとに初期化漏れを検出できるか確認しているか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: Part 1〜3 の本文拡充後にチェックリストへ反映する。

> 対応元: P2 / Part 1〜3 の本文拡充後にチェックリストへ反映する。

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
