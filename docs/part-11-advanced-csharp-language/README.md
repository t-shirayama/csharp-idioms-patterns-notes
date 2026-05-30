# Part 11: C# 言語機能のステップアップ

この Part では、現代 C# の基礎を越えて、実務コードで設計の表現力を上げる言語機能を扱います。対象は generics、variance、delegate、event、extension method、attribute、iterator、`yield`、primary constructor などです。

ここでの目的は「高度な構文を使えるようになること」ではありません。型安全性、拡張性、責務分離、テストしやすさ、レビューしやすさを損なわずに、C# の表現力をどう使うかを判断できるようにすることです。

## このPartで身につくこと

- generics と constraints を、重複削減ではなく型契約の表現として使える。
- covariance / contravariance を、コレクション、delegate、interface の代入互換性として説明できる。
- delegate、event を、コールバック、通知、疎結合化の手段として使い分けられる。
- extension method を、既存型の補助APIとして安全に追加できる。
- attribute を、メタデータ、フレームワーク連携、source generator との接点として読める。
- iterator と `yield` を、遅延評価、ストリーミング、リソース管理の観点でレビューできる。
- primary constructor などの新しい構文を、簡潔さと読みやすさのバランスで採用できる。

## 読む順番

1. [generics、variance、constraints](generics-variance-constraints.md) で、型パラメータにどこまで契約を持たせるかを整理する。
2. [delegates、events](delegates-events.md) で、コールバックと通知の責務境界を確認する。
3. [extension methods](extension-methods.md) で、既存型を汚さずに読みやすい補助APIを足す方法を押さえる。
4. [attributes](attributes.md) で、実行時メタデータとビルド時処理の境界を理解する。
5. [iterators、yield](iterators-yield.md) で、遅延評価と例外発生タイミングをレビューできるようにする。
6. [primary constructors and modern syntax](primary-constructors-and-modern-syntax.md) で、新しい構文を使う判断基準を固める。

最後に [コードレビューチェックリスト](checklist.md) を使い、Part 11 の観点で自分のコードを見直します。

## 重要トピック

- generics は「何でも受け取れるようにする」ためではなく、同じ振る舞いを型安全に共有するために使う。
- constraints は実装都合だけでなく、呼び出し側が期待してよい能力を示す契約でもある。
- delegate と event は似ていますが、event は発火側の所有権を守る公開方法として読む。
- extension method は便利な一方、発見しづらい依存や責務の散逸を生みやすい。
- attribute は宣言的で読みやすい反面、実行時に何が起きるかがコードだけでは見えにくい。
- `yield` はメモリ効率を上げられますが、評価タイミング、例外、dispose のタイミングが遅れる。
- 新構文は短さより、既存チームが読んで意図を追えるかを優先する。

## 実務での使いどころ

共通部品、ライブラリ、フレームワーク寄りのコード、拡張ポイントを持つアプリケーションで特に役立ちます。たとえば、型ごとの重複した変換処理を generics でまとめる、ドメインイベントやUI通知を event で表す、テスト用のデータ生成補助を extension method にする、validation やシリアル化の設定を attribute で宣言する、といった場面です。

一方で、アプリケーションの業務ロジックにこれらを過剰に入れると、処理の流れが見えにくくなります。高度な言語機能は、責務の境界を明確にするために使い、単に短く書くためだけには使わないことが重要です。

## よくある落とし穴

- generics を使いすぎて、具体的な業務ルールが型パラメータの奥に隠れる。
- constraints が弱すぎて実装内で reflection や unsafe なキャストに逃げる。
- event の購読解除を忘れ、長寿命オブジェクトから短寿命オブジェクトが解放されない。
- extension method が増えすぎて、どの namespace を import すると何が使えるのか分からなくなる。
- attribute に処理の本体を寄せすぎ、テストやデバッグが難しくなる。
- `yield` で返した sequence を複数回列挙し、I/O や重い計算が繰り返される。
- primary constructor や collection expression を、チームの読みやすさを確認せず一気に広げる。

## レビュー観点

- generic type / method は、型パラメータが本当に必要な抽象化になっているか。
- constraints は、実装の前提と呼び出し側への契約を過不足なく表しているか。
- delegate / event は、発火側、購読側、解除責任、例外の扱いが明確か。
- extension method は、対象型の自然な語彙として読めるか。業務ロジックを隠していないか。
- attribute による挙動は、テスト、ドキュメント、起動時検証のいずれかで確認できるか。
- `IEnumerable<T>` を返す処理は、遅延評価であることが呼び出し側にとって自然か。
- 新構文の採用は、既存コードのスタイル、対象 C# バージョン、チームの理解と揃っているか。

## 記事一覧

- [generics、variance、constraints](generics-variance-constraints.md): 型パラメータ、制約、共変性、反変性を設計判断として扱います。
- [delegates、events](delegates-events.md): コールバック、通知、イベント購読の責務と落とし穴を整理します。
- [extension methods](extension-methods.md): 既存型への補助API追加を、読みやすさと責務境界の観点で扱います。
- [attributes](attributes.md): 属性をメタデータ、フレームワーク連携、テスト容易性の観点で確認します。
- [iterators、yield](iterators-yield.md): 遅延評価、ストリーミング、例外と dispose のタイミングを整理します。
- [primary constructors and modern syntax](primary-constructors-and-modern-syntax.md): primary constructor などの新構文を採用する判断基準をまとめます。
- [コードレビューチェックリスト](checklist.md): Part 11 の内容をレビュー時に確認するためのチェックリストです。

---

[章別ノート一覧に戻る](../index.md)
