# その他の実務定石

この補足編では、日々の業務アプリケーションで主役になりにくいものの、境界処理や運用時の不具合に直結しやすい定石を扱います。
プロセス起動、環境変数、バージョン情報、ZIP、reflection、source generator、unsafe / native interop は、どれも「少し書ける」だけでは足りず、失敗時の見え方、テストしやすさ、保守範囲を意識して使う必要があります。

## 項目一覧

- [補足トピックのレビュー checklist](checklist.md)
- [プロセス起動](プロセス起動.md)
- [環境変数](環境変数.md)
- [バージョン情報](バージョン情報.md)
- [ZIP 操作](zip-操作.md)
- [reflection](reflection.md)
- [source generator の入口](source-generator-の入口.md)
- [unsafe / native interop の入口](unsafe-native-interop-の入口.md)

## 読み返し方

- 実装前は各トピックの本文で、どこに境界を置くかを確認する。
- レビュー時は checklist を使い、入力、失敗時の見え方、テスト境界、運用差分を短い項目で確認する。
- プロセス起動、環境変数、ZIP 展開、reflection、source generator、unsafe / native interop は、正常系よりも「暗黙の前提」が不具合になりやすい。レビューでは、その前提がコードかテストか運用メモに残っているかを見る。

---

[章別ノート一覧に戻る](../README.md)

