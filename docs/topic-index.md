# 目的別索引

実務で迷いやすい目的から、関連する章へ直接移動するための索引です。

## 安全な外部 I/O

- [プロセス起動](appendix/プロセス起動.md): 子プロセス終了、標準出力、環境変数、引数マスク。
- [ZIP 操作](appendix/zip-操作.md): ZIP bomb、path traversal、同名衝突、atomic move。
- [一時ファイルと作業ディレクトリ](appendix/一時ファイルと作業ディレクトリ.md): cleanup、権限、並列実行、atomic replace。
- [file / path / directory 操作](part-05-strings-dates-files-data/file-path-directory-操作.md): パス正規化、ディレクトリ境界、ファイル操作。

## 設定、secret、運用

- [環境変数](appendix/環境変数.md): ASP.NET Core configuration、階層キー、scope、reload 不可。
- [configuration と secret operations](part-15-operations-quality-delivery/configuration-and-secret-operations.md): secret scanning、ログマスク、Key Vault、ローテーション。
- [health checks / readiness / liveness](part-15-operations-quality-delivery/health-checks-readiness-liveness.md): probe、degraded、依存先別 readiness。

## 配布物、バージョン、互換性

- [バージョン情報](appendix/バージョン情報.md): SemVer、AssemblyVersion、commit hash、container tag。
- [NuGet / versioning / library design](part-15-operations-quality-delivery/nuget-versioning-library-design.md): pack、署名、API compatibility、Source Link。
- [trimming / NativeAOT 互換性](appendix/trimming-nativeaot-互換性.md): reflection、JSON source generation、DI、native interop。

## メタプログラミングと native 境界

- [reflection](appendix/reflection.md): trimming 属性、NullabilityInfoContext、AssemblyLoadContext。
- [source generator の入口](appendix/source-generator-の入口.md): incremental generator、diagnostic、テスト、packaging。
- [unsafe / native interop の入口](appendix/unsafe-native-interop-の入口.md): LibraryImport、marshaling、pinning、endianness。

## レビュー導線

- [全チェックリスト導線](checklists.md)
- [コードレビュー観点](part-13-style-review-maintainability/コードレビュー観点.md)
- [analyzer、EditorConfig、nullable 警告との付き合い方](part-13-style-review-maintainability/analyzer-editorconfig-nullable-警告との付き合い方.md)
