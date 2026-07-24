# Swift版 vi/ex 再実装 — 技術調査サマリー

作成日: 2026-07-24
対象リポジトリ: https://github.com/murakami/vi (ex/vi 3.7 の C言語ポート)

このドキュメントは、既存のC実装(ex/vi 3.7 + BSD termcap由来)をSwiftで再実装するにあたって
事前に調査・検討した内容の引き継ぎメモです。Claude Codeでこのリポジトリ配下の作業を始める際の
コンテキストとして使ってください。

---

## 1. 元リポジトリの性質

- 6/7/85版 ex/vi 3.7 と 2.11BSD termcapライブラリ由来のポート
- 対象は伝統的なUNIX系(Linux, Solaris, HP-UX, AIX, Tru64, SCO UnixWare, FreeBSD, NetBSD等)
- Windowsは明示的に対象外(README記載)
- 多バイトロケール対応(UTF-8, EUC-JP, EUC-KR, Big5, Big5-HKSCS, GB2312, GBK)を自前実装
- 独自malloc/mallocラッパー(`malloc.c`, `mapmalloc.c`)、独自`printf.c`、`sbrk()`によるヒープ拡張など
  1985年当時のUNIX環境差異を吸収するための実装が多くを占める
- `catd`ディレクトリ: 伝統的なメッセージカタログ(catgets系)によるローカライズ機構

## 2. 今回のSwift版のスコープ決定事項(会話で確定した方針)

| 項目 | 方針 |
|---|---|
| 対象プラットフォーム | **macOS / Linux / FreeBSD** の3つに限定。Solaris/AIX/HP-UX等の商用UNIXはSwiftツールチェーン自体が存在しないため対象外 |
| 移植方針 | 逐語移植ではなく「原典を仕様書として読み、Swiftらしく作り直す」。malloc/sbrk独自実装・独自printfなどC時代の環境差異吸収コードは持ち込まない |
| 文字コード | Swiftの`String`(Unicode正規, grapheme cluster単位)にフルベット。手動UTF-8/EUC-JP/Big5デコードは不要。UTF-8前提で一本化 |
| 表示幅計算 | 文字コードとは別問題として残る。`wcwidth()`相当が必要な箇所は`Unicode.Scalar.properties.isEastAsianWide`または自前テーブルで対応(Glibcの`wcwidth`呼び出しは不要) |
| 正規表現 | Swift標準の`Regex`型(Swift 5.7+)を使用。原典の`ex_re.c`が実装する伝統的BRE/ERE(`\(...\)`グルーピング、`\<`/`\>`単語境界等)との挙動差異は許容する、という判断済み。動的パターン(`:s/foo/bar/`等ユーザー入力起点)は`try Regex(pattern)`で都度コンパイル |
| ローカライズ | 伝統的なUNIX機構(catgets/`catd`)ではなく、Swift Package Managerの標準機構を使用。`Package.swift`に`defaultLocalization`を指定し、ターゲット内に`<lang>.lproj/Localizable.strings`を配置、`Bundle.module`経由で参照する方式。これはコマンドラインexecutableターゲットでも問題なく機能することを確認済み。String Catalogs(`.xcstrings`)はSPMがネイティブ対応していないため今回は見送り、まず`.strings`方式で開始する |
| ターミナル制御 | 残る唯一のプラットフォーム依存箇所。ncursesをsystem-libraryターゲットで包む方式を採用 |

## 3. ターミナル制御層の実装方針

原典の独自termcapライブラリ(`libterm`)は、ncursesベースの薄いラッパーに置き換える。

**注意点(macOSとLinux/FreeBSDでの扱いの違い)**
- macOS: Darwin moduleに標準でncursesが同梱されているため `import Darwin` 経由でそのまま使える
- Linux / FreeBSD: システムのncursesをsystem-libraryターゲット(例: `CNcurses`)でラップする必要がある。
  Package.swiftの`providers`に`apt`(Linux)や`brew`/`pkg`(FreeBSD相当)を指定するパターンが定番
- macOS側のDarwin moduleが独自にncursesヘッダを取り込んでいるため、Linux用に書いたmodulemapを
  そのままmacOSに適用するとヘッダの二重定義でビルドエラーになる。
  `#if canImport(Darwin)` / `#else` で経路を分岐させる必要がある

参考実装パターン: `rderik/Cncurses`(system-libraryターゲットの最小構成例)

## 4. 提案パッケージ構成

```
Package.swift          ← defaultLocalization指定、platforms: [.macOS(...)] (Linux/FreeBSDは暗黙対応)
Sources/
  ExCore/               ← 純Swift。テキストバッファ、コマンドディスパッチ、
                           アドレッシング(ex_addr.c相当)。OS依存ゼロ。
                           文字コード=String、正規表現=Regex型を直接利用
  Terminal/             ← termios制御 + ncurses呼び出しの薄い抽象化層。
                           #if canImport(Darwin) でmacOS/Linux/FreeBSDを吸収
  CNcurses/              ← system-libraryターゲット(Linux/FreeBSD用ncursesラッパー)
  vi/                    ← executableターゲット(エントリポイント)
Sources/vi/Resources/
  en.lproj/Localizable.strings
  ja.lproj/Localizable.strings
Tests/
  ExCoreTests/           ← コマンド互換性のゴールデンテスト(原典のex.1/vi.1マニュアルを仕様として)
```

設計の要点: `ExCore`にロジックの大半を集約しOS依存ゼロに保つ。プラットフォーム差異は`Terminal`
ターゲット1箇所に閉じ込める。原典のようなシステムごとのMakefile分岐は不要になる想定。

## 5. 実装の進め方(提案)

1. `ex.1` / `vi.1` マニュアルページと `TODO` / `Changes` ファイルをコマンド仕様の一次情報として棚卸し
2. `ExCore`(バッファ管理・コマンド処理)をプラットフォーム非依存で先に実装し、ユニットテストで固める
3. Linux上でncurses経由の最小viループを動作確認
4. macOS対応を `#if canImport(Darwin)` 分岐で追従
5. FreeBSD動作確認
6. 正規表現の互換性差異(BRE/ERE vs Swift Regex)、東アジア文字幅計算などUNIX間で差が出やすい部分は
   最後に個別検証

## 6. 未検討・要フォローアップの論点

- `ex_addr.c` のアドレス指定構文(`.`, `$`, `%`, 行範囲指定など)をSwiftの型(enum + パーサーコンビネータ等)
  でどう表現するか、具体的な設計は未着手
- テキストバッファの内部データ構造(gap buffer / piece table等)の選定は未検討
- `:s`コマンド用の動的正規表現コンパイルのエラーハンドリング方針は未検討
- FreeBSD上でのSwiftツールチェーンの安定性(コミュニティ提供、公式ではない)は実機検証が必要
- Androidや他プラットフォームへの展開は今回のスコープ外(過去の別調査で概要は把握済み: Swift 6.3で
  公式Android SDKが追加されたが、これは今回のvi再実装には直接関係しない)

## 7. 前提として不要と判断した検討事項(参考)

以下は当初検討したが、今回のスコープ決定により不要と判断した内容:

- POSIX regex.h(`regcomp`/`regexec`)をsystem-libraryターゲットで直接呼ぶ案 → Swift Regexを使う方針
  に確定したため不採用
- 独自wcwidth実装のためのGlibc/Darwin `wcwidth()` 呼び出し → Unicode.Scalar propertiesベースの
  自前実装で代替する方針
- catgets/メッセージカタログ形式のローカライズ移植 → SPM標準のlproj方式に置き換え
