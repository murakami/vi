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
| 対象プラットフォーム | **macOS / Linux / FreeBSD** の3つに限定。Solaris/AIX/HP-UX等の商用UNIXはSwiftツールチェーン自体が存在しないため対象外。ただしFreeBSDは設計上の考慮対象とするのみで、実機での動作確認は行わない(スコープ外) |
| 移植方針 | 逐語移植ではなく「原典を仕様書として読み、Swiftらしく作り直す」。malloc/sbrk独自実装・独自printfなどC時代の環境差異吸収コードは持ち込まない |
| 文字コード | 内部表現はSwiftの`String`(Unicode正規, grapheme cluster単位)に統一し、`ExCore`のロジックはUTF-8前提のまま変更しない。ただしファイルI/Oの境界層で`Foundation`の`String(data:encoding:)`/`data(using:)`(`.japaneseEUC`, `.shiftJIS`等)を使い、EUC-JP/Shift-JIS/Big5等のレガシーエンコーディングにも対応する。手動デコードの自前実装は行わない。エンコーディングの自動判定は信頼できないため、原典CのLANG依存にならい、ロケール環境変数または`:set encoding=`相当の明示指定で決定する方式とする。macOS/Linuxは`FoundationInternationalization`のICU統合で問題なく動く見込みだが、FreeBSDについては設計上考慮するに留め、実機での動作確認は行わない(§6参照) |
| 表示幅計算 | 文字コードとは別問題として残る。`wcwidth()`相当が必要な箇所は`Unicode.Scalar.properties.isEastAsianWide`または自前テーブルで対応(Glibcの`wcwidth`呼び出しは不要) |
| 正規表現 | Swift標準の`Regex`型(Swift 5.7+)を使用。原典の`ex_re.c`が実装する伝統的BRE/ERE(`\(...\)`グルーピング、`\<`/`\>`単語境界等)との挙動差異は許容する、という判断済み。動的パターン(`:s/foo/bar/`等ユーザー入力起点)は`try Regex(pattern)`で都度コンパイル |
| ローカライズ | 原典も`catd/`に`en_US`カタログしかなく、`Makefile`の`LANGMSG`(catgets有効化フラグ)はデフォルトで無効。つまり原典も実運用では英語メッセージのみであり、これはスコープを削るものではなく原典の実際の挙動への忠実な選択。当面はメッセージ文字列をSwiftソースに直接埋め込み(英語のみ)、SwiftPMのリソースバンドル(`.lproj`/`Bundle.module`)は導入しない。理由: SPMのリソース機構を使うと実行ファイルと別に`.bundle`リソースディレクトリが生成され、両者をセットで配置する必要が生じ、原典が持っていた「単一バイナリ配布」の性質が崩れるため。複数言語対応の実需が生じた時点で、`Package.swift`に`defaultLocalization`を指定し`<lang>.lproj/Localizable.strings`+`Bundle.module`経由の参照に切り替える(SPM 5.3以降が`.lproj`構成に対応済みであることは確認済み。`.process()`ルールが必要)。String Catalogs(`.xcstrings`)はSPMがネイティブ対応していないため、切り替える場合も`.strings`方式を使う |
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
Package.swift          ← platforms: [.macOS(...)] (Linux/FreeBSDは暗黙対応)。
                           defaultLocalizationは現時点では不要(下記参照)
Sources/
  ExCore/               ← 純Swift。テキストバッファ、コマンドディスパッチ、
                           アドレッシング(ex_addr.c相当)。OS依存ゼロ。
                           文字コード=String、正規表現=Regex型を直接利用
                           メッセージ文字列は当面英語のみでSwiftソースに直接埋め込む
  Terminal/             ← termios制御 + ncurses呼び出しの薄い抽象化層。
                           #if canImport(Darwin) でmacOS/Linux/FreeBSDを吸収
  CNcurses/              ← system-libraryターゲット(Linux/FreeBSD用ncursesラッパー)
  vi/                    ← executableターゲット(エントリポイント)
Tests/
  ExCoreTests/           ← コマンド互換性のゴールデンテスト(原典のex.1/vi.1マニュアルを仕様として)
```

(複数言語対応が必要になった場合は`Sources/vi/Resources/<lang>.lproj/Localizable.strings`+
`Package.swift`への`defaultLocalization`追加+`Bundle.module`参照へ切り替える。単一バイナリ配布を
維持するため、それまではこの構成に着手しない)

設計の要点: `ExCore`にロジックの大半を集約しOS依存ゼロに保つ。プラットフォーム差異は`Terminal`
ターゲット1箇所に閉じ込める。原典のようなシステムごとのMakefile分岐は不要になる想定。

文字コード変換層(EUC-JP等↔UTF-8)は`ExCore`ではなく、ファイルの読み書きを担う`vi`実行ターゲット側
(またはそのための小さな`FileIO`層)に置き、`Foundation`への依存をそこだけに閉じ込める。`ExCore`は
常にデコード済みのSwift `String`だけを扱い、エンコーディングの知識を一切持たない。

## 4.5 テキストバッファのデータ構造 — オリジナルの設計からの示唆

原典`ex_temp.c`(詳細は`docs/ex-mode.md`§4)の巨大ファイル耐性は、「行ポインタの配列(コア上)」
+「実テキストを格納する追記専用の一時ファイル(ディスク上)」という二層構造に由来する。

- 行ごとに保持するのは一時ファイル内オフセットをpackingした数バイトの整数のみで、テキスト本体は
  常にディスク側に置く。メモリ使用量はファイルの**バイトサイズ**ではなく**行数**に比例するため、
  巨大ファイルでも実用的な速度で動作していた。
- `getline()`/`putline()`が3枚だけのブロックキャッシュ経由でディスクとの間を1行単位で出し入れする。
- 書き込みは常に追記のみで、上書き・GCは意図的に省略している(undo/マークの整合性を保つ実装コストを
  避けるための割り切り。コメントに明記あり)。副作用として編集するほど一時ファイルは肥大化し続ける。

この設計は本質的にpiece table(原本ファイル+追記専用の編集ログという2つのソースを持つ)の前身にあたる。
Swift版のテキストバッファは、この「行粒度でディスク/mmapにバックエンドを持ち、メモリ使用量を行数に
比例させる」という性質を引き継ぐ形でpiece table方式を採用する方針とする。ただし以下の点は原典の
実装をそのまま踏襲しない:

- 原典は1行を生バイト列のまま一時ファイルに置き、マルチバイト処理は上位レイヤーで行っている。
  Swift版は内部表現をデコード済み`String`に統一する方針(§2「文字コード」参照)のため、行インデックスが
  指す先は「原本ファイルのmmap範囲、または追記ログ内のデコード済みテキストへの参照」とし、生バイト
  オフセットのpackingという原典の実装詳細は移植しない。
- GC省略という原典の割り切りは、現代的なpiece tableであればコンパクション機構を持たせられるため、
  そのまま踏襲する必然性はない(ただし1セッションの対話的編集用途では、原典同様GCなしでも実用上
  問題にならない可能性が高く、実装コストとのバランスで判断する)。

## 5. 実装の進め方(提案)

1. `ex.1` / `vi.1` マニュアルページと `TODO` / `Changes` ファイルをコマンド仕様の一次情報として棚卸し
2. `ExCore`(バッファ管理・コマンド処理)をプラットフォーム非依存で先に実装し、ユニットテストで固める
3. Linux上でncurses経由の最小viループを動作確認
4. macOS対応を `#if canImport(Darwin)` 分岐で追従
5. 正規表現の互換性差異(BRE/ERE vs Swift Regex)、東アジア文字幅計算などUNIX間で差が出やすい部分は
   最後に個別検証

(FreeBSDは`#if`分岐上は考慮するが、実機での動作確認ステップは今回の進め方に含めない)

## 6. 未検討・要フォローアップの論点

- `ex_addr.c` のアドレス指定構文(`.`, `$`, `%`, 行範囲指定など)をSwiftの型(enum + パーサーコンビネータ等)
  でどう表現するか、具体的な設計は未着手
- テキストバッファの内部データ構造は、原典`ex_temp.c`の設計(§4.5参照)を踏まえてpiece table方式に
  方向性は決めたが、具体的な型設計(mmap範囲・編集ログ・行インデックスの3者の具体的なSwift表現)は未着手
- `:s`コマンド用の動的正規表現コンパイルのエラーハンドリング方針は未検討
- FreeBSD上でのSwiftツールチェーンの安定性(コミュニティ提供、公式ではない)、および
  `FoundationInternationalization`(ICU統合)がFreeBSD向けにビルド・動作するかは未確認のまま残る。
  今回のスコープでは設計上の考慮(`#if`分岐での対応)に留め、実機検証は行わない方針とした
- レガシーエンコーディング(EUC-JP, Shift-JIS, Big5等)からUnicodeへの変換における非可逆文字(ベンダー
  拡張漢字、半角/全角の重複マッピング等)の扱い、および保存時のラウンドトリップ検証方針は未検討
- Androidや他プラットフォームへの展開は今回のスコープ外(過去の別調査で概要は把握済み: Swift 6.3で
  公式Android SDKが追加されたが、これは今回のvi再実装には直接関係しない)

## 7. 前提として不要と判断した検討事項(参考)

以下は当初検討したが、今回のスコープ決定により不要と判断した内容:

- POSIX regex.h(`regcomp`/`regexec`)をsystem-libraryターゲットで直接呼ぶ案 → Swift Regexを使う方針
  に確定したため不採用
- 独自wcwidth実装のためのGlibc/Darwin `wcwidth()` 呼び出し → Unicode.Scalar propertiesベースの
  自前実装で代替する方針
- catgets/メッセージカタログ形式のローカライズ移植 → 原典もデフォルトでは英語のみのため不採用。
  SPM標準の`.lproj`+`Bundle.module`方式も、単一バイナリ配布を崩さないため当面は導入せず、
  英語メッセージをSwiftソースに直接埋め込む方式とした(§2「ローカライズ」参照。将来必要になれば
  `.lproj`方式へ切り替え可能)
