# interface と abstract class の使い分け

## 概要

`interface` は「何ができるか」という契約を表し、`abstract class` は共通状態や共通アルゴリズムを含む基底型を表します。テストダブルのために interface を作る場面はありますが、すべてのクラスに機械的に interface を付けると設計の見通しが悪くなります。

## 使いどころ

- `interface`: 複数実装、外部 I/O の差し替え、アプリケーション層からインフラ層への依存反転。
- `abstract class`: 派生型に共通の状態、保護された補助メソッド、固定した処理順序がある。
- default interface methods は、公開 API の互換性維持など限定的な用途にとどめる。
- `interface` は呼び出し側が必要とする最小の能力として定義し、実装クラスの都合を持ち込まない。

```csharp
public interface IInvoiceNumberGenerator
{
    string Generate(DateOnly issuedOn);
}

public sealed class MonthlyInvoiceNumberGenerator : IInvoiceNumberGenerator
{
    public string Generate(DateOnly issuedOn) => $"{issuedOn:yyyyMM}-001";
}
```

## 判断基準

まず「呼び出し側は何をしたいのか」を名前にできるなら `interface` が向きます。例えば `IFileSystem` のような大きな名前より、呼び出し側に必要な `IReportStore` のほうがテストでも本番でも意味が保ちやすくなります。

一方で、共通の状態や処理順序を派生型に強制したいなら `abstract class` を検討します。ただし、基底クラスが大きくなるほど派生型の自由度は下がるため、公開する `protected` メンバーは最小にします。

```csharp
public abstract class CsvImporter
{
    public async Task ImportAsync(Stream stream, CancellationToken cancellationToken)
    {
        using var reader = new StreamReader(stream);

        while (await reader.ReadLineAsync(cancellationToken) is { } line)
        {
            var row = Parse(line);
            Validate(row);
            await SaveAsync(row, cancellationToken);
        }
    }

    protected abstract CsvRow Parse(string line);
    protected abstract void Validate(CsvRow row);
    protected abstract Task SaveAsync(CsvRow row, CancellationToken cancellationToken);
}
```

## 避けたい書き方

- 実装が 1 つしかなく、差し替え予定もないのに `IFoo` を量産する。
- `interface` にプロパティを増やしすぎ、実装クラスが不要なメンバーまで背負う。
- `abstract class` の protected 状態に派生型が強く依存し、変更時の影響範囲が読めない。
- `IRepository<T>` のような汎用 interface に寄せすぎて、業務ごとの制約や意図が消える。

```csharp
// 避けたい例: 参照しかしない処理まで Save/Delete に依存する。
public interface IRepository<T>
{
    Task<T?> FindAsync(Guid id, CancellationToken cancellationToken);
    Task SaveAsync(T entity, CancellationToken cancellationToken);
    Task DeleteAsync(Guid id, CancellationToken cancellationToken);
}
```

## よくある失敗

- テストダブルを作りたいだけで interface を切り、実際の設計境界が説明できない。
- `interface` を細かく分けすぎて、呼び出し側のコンストラクタ引数が増えすぎる。
- `abstract class` のコンストラクタや初期化順序が複雑で、派生型の実装ミスが実行時まで分からない。
- default interface methods に実装を寄せ、interface が事実上の基底クラスになる。

## レビュー観点

- その抽象化は呼び出し側の都合を表しているか、実装側の都合を漏らしていないか。
- テストのための interface が、本番コードでも意味のある境界になっているか。
- 公開済み interface にメンバーを追加して、既存実装を壊さないか。
- 継承階層が深くなり、振る舞いの追跡が難しくなっていないか。
- abstract class を選んだ場合、派生型が守るべき順序や不変条件をテストできているか。

---

[Part README に戻る](README.md)

<!-- TODO対応追記 -->

## TODO対応: interface を port として使う場合

> 対応元: P1 / interface を port として使う場合、公開 API と内部 interface の違い、interface 追加・変更時の互換性、default interface method の注意を拡充する。

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
