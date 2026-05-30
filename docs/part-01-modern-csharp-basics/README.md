# Part 1: 現代 C# の基礎体力

この Part では、C# の文法を細かく覚える前に、実務コードを読む・書く・レビューするための土台を整理します。対象は C# 8 以降を中心に、nullable reference types、`record`、`init`、`required`、file-scoped namespace など、現代 C# で判断を間違えやすい領域です。

ここでの目的は「機能名を知ること」ではなく、プロジェクトの前提、型の選び方、初期化の責任、例外とリソース管理を見たときに、設計上の意図を説明できるようにすることです。

## このPartで身につくこと

- C# と .NET の関係、Target Framework、SDK、実行環境の違いを説明できる。
- 値型、参照型、nullable reference types を、null 安全性とAPI設計の観点で使い分けられる。
- `class`、`struct`、`record`、`record struct` を、同一性、等価性、変更可能性で選べる。
- `property`、`init`、`required`、`readonly` を、初期化漏れや不変条件の表現に使える。
- namespace と `using` を、読みやすさと衝突回避の観点で整理できる。
- 例外、`using`、`IDisposable` を、リソースリークや責務境界の観点でレビューできる。

## 読む順番

1. [C# と .NET の全体像](csharp-と-dotnet-の全体像.md) で、言語、ランタイム、ライブラリ、プロジェクト設定の関係を押さえる。
2. [型、値型、参照型、nullable reference types](型-値型-参照型-nullable-reference-types.md) で、null と型選択の判断基準を固める。
3. [class / struct / record / record struct](class-struct-record-record-struct.md) で、データの性質に合う型表現を選ぶ。
4. [property、init、required、readonly](property-init-required-readonly.md) で、初期化と変更可能性をコードに表す。
5. [namespace、using、file-scoped namespace](namespace-using-file-scoped-namespace.md) で、ファイル構造と名前解決を整える。
6. [exception、using、IDisposable、リソース管理](exception-using-idisposable-リソース管理.md) で、失敗時と解放時の責任を明確にする。

最後に [コードレビューチェックリスト](checklist.md) を使い、Part 1 の観点で自分のコードを見直します。

## 重要トピック

- nullable reference types は「警告を消す仕組み」ではなく、API契約を型に寄せるための仕組みとして扱う。
- `record` はDTOやValue Objectに向きますが、ライフサイクルやIDを持つEntityには慎重に使う。
- `required` は便利ですが、バリデーションや不変条件の代替ではない。
- `struct` は値型にしたい理由だけで選ばず、サイズ、コピーコスト、default 値、変更可能性まで見る。
- `IDisposable` は「使ったら閉じる」だけでなく、所有権の境界を明確にするための設計要素として読む。

## 実務での使いどころ

新規プロジェクトの雛形を読むとき、既存コードの型設計をレビューするとき、DTO、Entity、設定クラス、オプション値、リソース管理を設計するときに使います。特に nullable を有効化したプロジェクトでは、`?`、`required`、コンストラクター、プロパティ初期化の組み合わせが、そのままAPIの使いやすさに影響します。

また、古い C# で書かれたコードを現代化するときにも、この Part の観点が役立ちます。単に新機能へ置き換えるのではなく、null 契約、変更可能性、所有権、例外の意味を崩さないことが重要です。

## よくある落とし穴

- nullable 警告を `!` で黙らせ、実行時の null 事故を残す。
- DTO、Entity、Value Object をすべて `class` またはすべて `record` で統一してしまう。
- `required` を付けたことで、不正な値まで防げたと誤解する。
- mutable な `struct` をコレクションやプロパティ越しに扱い、変更が反映されないバグを生む。
- `using` や `IDisposable` の責任を呼び出し側と生成側のどちらが持つか曖昧にする。

## レビュー観点

- public API の null 許容性は、呼び出し側にとって自然で一貫しているか。
- 型の選択は、同一性、等価性、変更可能性、コピーコストを説明できるか。
- 初期化必須の値は、コンストラクター、`required`、ファクトリのどれで表すべきか整理されているか。
- 例外は握りつぶされず、境界で意味のある形に変換されているか。
- リソースを所有する型は、解放責任とライフサイクルが読み取れるか。

## 記事一覧

- [C# と .NET の全体像](csharp-と-dotnet-の全体像.md): C#、.NET、SDK、Target Framework、プロジェクト構成の関係を整理します。
- [型、値型、参照型、nullable reference types](型-値型-参照型-nullable-reference-types.md): 型の基本と nullable を、API契約とレビュー観点から扱います。
- [class / struct / record / record struct](class-struct-record-record-struct.md): 同一性、等価性、変更可能性に基づいて型を選ぶ判断基準をまとめます。
- [property、init、required、readonly](property-init-required-readonly.md): 初期化、読み取り専用、不変条件をプロパティ設計に落とし込む方法を扱います。
- [namespace、using、file-scoped namespace](namespace-using-file-scoped-namespace.md): 名前空間と using を、ファイル構造と読みやすさの観点で整理します。
- [exception、using、IDisposable、リソース管理](exception-using-idisposable-リソース管理.md): 例外設計、using 文、リソース解放、所有権の境界を確認します。
- [コードレビューチェックリスト](checklist.md): Part 1 の内容をレビュー時に確認するための短いチェックリストです。

---

[章別ノート一覧に戻る](../index.md)
