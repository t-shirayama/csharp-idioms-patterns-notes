# Part 5: オブジェクト指向と設計パターン

オブジェクト指向や設計パターンは、クラスを増やすための技法ではなく、変更理由を分け、依存の向きを整え、テストしやすい境界を作るための道具です。

C# では `class`、`interface`、`abstract class`、`record`、DI コンテナなど選択肢が多いため、「できる」よりも「この場面で選ぶ理由があるか」を重視します。

## 読み方

この Part では、オブジェクト指向を「継承を使うこと」ではなく、状態、振る舞い、依存関係をどこに置くかの判断として扱います。設計パターンや SOLID 原則は、名前を覚えることよりも、変更理由を説明できるか、テストで境界を押さえられるかを確認するための道具です。

- 状態を守る必要があるなら、まずカプセル化を確認する。
- 実装差し替えが必要なら、interface、Strategy、DI のどれが自然かを比べる。
- 共通処理をまとめたいだけなら、継承より合成や委譲で済まないかを考える。
- 外部 API や DB と接する型は、DTO、Entity、Value Object の境界を混ぜない。
- 原則やパターンを使ったあと、呼び出し側のコードが読みやすくなったかを見直す。

## 項目一覧

- [カプセル化、継承、ポリモーフィズム](カプセル化-継承-ポリモーフィズム.md)
- [interface と abstract class の使い分け](interface-と-abstract-class-の使い分け.md)
- [Template Method、Strategy、Factory](template-method-strategy-factory.md)
- [Dependency Injection](dependency-injection.md)
- [Value Object、Entity、DTO](value-object-entity-dto.md)
- [SOLID 原則を C# コードで読む](solid-原則を-csharp-コードで読む.md)

---

[章別ノート一覧に戻る](../README.md)

