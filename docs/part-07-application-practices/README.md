# Part 7: 実務アプリケーションの定石

アプリケーションコードでは、文法の正しさだけでなく、設定、ログ、入力検証、エラー、層の責務、API 境界、DB アクセスの扱いが保守性を大きく左右します。

この Part では、ASP.NET Core などの現代的な C# アプリケーションでよく出る判断を、レビューで確認しやすい単位に整理します。重要なのは「どこに書けるか」ではなく、「どの責務をどこに置くと、変更や障害調査に強くなるか」です。

## 項目一覧

- [設定と Options pattern](設定と-options-pattern.md)
- [ログと構造化ログ](ログと構造化ログ.md)
- [validation](validation.md)
- [エラー処理と例外設計](エラー処理と例外設計.md)
- [repository / service / usecase 分離](repository-service-usecase-分離.md)
- [Web API での DTO、境界、マッピング](web-api-での-dto-境界-マッピング.md)
- [DB アクセス時の注意点](db-アクセス時の注意点.md)

---

[章別ノート一覧に戻る](../README.md)

