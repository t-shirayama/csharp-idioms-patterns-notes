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


