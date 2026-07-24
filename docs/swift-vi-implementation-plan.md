# Swift版 vi/ex 再実装 — 実装計画

対象リポジトリ: https://github.com/murakami/vi
関連文書: [swift-vi-research-notes.md](swift-vi-research-notes.md)(方針決定の背景・検討経緯)

このドキュメントは実装作業の進捗管理用チェックリストです。項目を完了したら`- [ ]`を
`- [x]`に変更してください(GitHub上でチェックボックスとして表示されます)。フェーズは
上から順に進める想定ですが、フェーズ内の項目は並行して進めても構いません。

---

## フェーズ0: 準備・一次情報の棚卸し

- [ ] `ex.1` / `vi.1` マニュアルページを読み、コマンド仕様一覧を作成する
- [ ] `TODO` / `Changes` ファイルを読み、既知の未解決事項・過去の変更意図を把握する
- [ ] コマンド互換性チェックリスト(ゴールデンテストの元ネタ)のドラフトを作成する
- [ ] `docs/ex-mode.md` / `docs/vi-mode.md` / `docs/regex.md` の内容と、上記コマンド仕様の対応関係を確認する

## フェーズ1: リポジトリディレクトリ構成の整理

方針(決定済み): オリジナルのC実装は`ex-vi/`サブディレクトリへそのままの形で移動し、
新しいSwift実装はリポジトリ直下にSwiftPMの標準配置(`Package.swift`, `Sources/`, `Tests/`)
で置く。Swiftツールチェーン(`swift build`/`swift test`、Xcode/VSCodeの自動認識)が
追加設定なしで機能することを優先する。

- [ ] オリジナルC実装一式(`ex.c`等のトップレベル`.c`/`.h`ファイル、`libterm/`、`libuxre/`、
  `catd/`、`Makefile`、`makeoptions`、`config.h`、`README`、`LICENSE`、`Changes`、`TODO`、
  `ex.spec`、`ex.1`/`vi.1`マニュアル等)を`ex-vi/`ディレクトリへ移動する
- [ ] 移動後、`ex-vi/`内で`make`によるビルドが従来どおり通ることを確認する
  (`libterm/`/`libuxre/`への相対パス参照が壊れていないか)
- [ ] `CLAUDE.md`のビルド手順・パス記述を`ex-vi/`前提に更新する
- [ ] `docs/ex-mode.md`等、既存の調査文書内のファイルパス言及について、`ex-vi/`配下に
  移動した後も参照として成立するか確認し、必要なら注記を加える
- [ ] リポジトリ直下に新しいトップレベル`README.md`を作成し、このリポジトリが
  「オリジナルC実装(`ex-vi/`)」と「Swift再実装(ルート直下)」を同居させたものである
  ことを説明する
- [ ] 上記のファイル移動をコミットする際、`git mv`を使い移動として履歴に記録されるようにする

## フェーズ2: パッケージ骨格

- [ ] `Package.swift` を作成する(`defaultLocalization`、`platforms: [.macOS(...)]`を設定。Linux/FreeBSDは暗黙対応)
- [ ] `ExCore` ターゲットの雛形を作成する(空実装でよい)
- [ ] `Terminal` ターゲットの雛形を作成する
- [ ] `CNcurses` system-libraryターゲットを作成する(Linux/FreeBSD向けncursesラッパー)
- [ ] `vi` executableターゲットの雛形を作成する
- [ ] Linux上で`CNcurses`経由のビルドが通ることを確認する
- [ ] macOS上で`import Darwin`経由のビルドが通ることを確認する(`#if canImport(Darwin)`分岐)

## フェーズ3: ExCore(コアロジック、OS依存ゼロ)

- [ ] テキストバッファの内部データ構造を設計する(方向性は決定済み: piece table方式。原典
  `ex_temp.c`の「行ポインタ配列+追記専用一時ファイル」構造を参考に、原本ファイルのmmap範囲/
  追記用編集ログ/行インデックスの3者をSwiftの型として具体化する。詳細は
  `swift-vi-research-notes.md`§4.5参照)
- [ ] 行アドレス指定構文(`ex_addr.c`相当: `.`, `$`, `%`, 行範囲, `/pat/`, `'m`等)をSwiftの型で設計する
- [ ] アドレス解析ロジックを実装する
- [ ] `:`コマンドディスパッチの仕組みを実装する(`ex_cmds.c`系のswitchディスパッチ相当)
- [ ] 正規表現ラッパーを実装する(Swift標準`Regex`型を使用、`try Regex(pattern)`による動的コンパイル)
- [ ] `:s///`用の動的正規表現コンパイルのエラーハンドリング方針を決定・実装する
- [ ] `ExCoreTests`を整備する(`ex.1`/`vi.1`を仕様としたゴールデンテスト)

## フェーズ4: 文字コード / FileIO層

- [ ] `FileIO`層(または`vi`ターゲット内モジュール)を設計する。`Foundation`への依存はここに閉じ込める
- [ ] エンコーディング決定方式を実装する(ロケール環境変数 または `:set encoding=`相当の明示指定。自動判定はしない)
- [ ] `Foundation`の`String(data:encoding:)`/`data(using:)`によるEUC-JP/Shift-JIS/Big5等の変換を実装する
- [ ] 非可逆文字(ベンダー拡張漢字、半角/全角の重複マッピング等)の扱い方針を決定する
- [ ] 主要エンコーディングについて読み込み→保存のラウンドトリップ検証を行う
- [ ] 表示幅計算を実装する(`Unicode.Scalar.properties.isEastAsianWide`または自前テーブル。`wcwidth()`は使わない)

## フェーズ5: Terminal制御層

- [ ] termios制御(raw modeの設定/解除)を実装する
- [ ] ncursesの薄いラッパーを実装する(`#if canImport(Darwin)` / `#else`でmacOSとLinux/FreeBSDを分岐)
- [ ] Linux上でncurses経由の最小viループ(起動・カーソル移動・簡単な編集・終了)を動作確認する
- [ ] macOS上で同等の最小viループを動作確認する

## フェーズ6: メッセージ文字列(当面は英語のみ、ローカライズ基盤は見送り)

方針(決定済み): 原典もデフォルトビルドでは英語メッセージのみ(`catd/`は`en_US`のみ、`LANGMSG`は
無効がデフォルト)なので、Swift版も当面は英語のみとする。SPMの`.lproj`/`Bundle.module`方式は
実行ファイルと別にリソースバンドルを生成し単一バイナリ配布を崩すため、多言語対応の実需が
出るまでは導入しない。詳細は`swift-vi-research-notes.md`§2「ローカライズ」参照。

- [ ] メッセージ文字列を集約する場所を`ExCore`/`vi`内に決める(定数 or enum。ハードコード英語)
- [ ] (将来・保留)複数言語対応が必要になった場合: `Package.swift`に`defaultLocalization`を追加し
  `Sources/vi/Resources/<lang>.lproj/Localizable.strings`+`Bundle.module`方式へ切り替える

## フェーズ7: 検証・仕上げ(FreeBSDは設計考慮のみ、実機検証は対象外)

- [ ] 正規表現の互換性差異(伝統的BRE/ERE の `\(...\)` グルーピング、`\<`/`\>` 単語境界等 vs Swift `Regex`)を個別検証する
- [ ] 東アジア文字幅計算の検証(全角/半角混在テキストでの表示崩れがないか)
- [ ] `#if canImport(Darwin)`分岐がFreeBSDのビルド構成上も破綻しないことをコードレビューで確認する(実機検証は行わない)
- [ ] 主要コマンドの一通りの動作確認(挿入・削除・置換・検索・アンドゥ・タグジャンプ等)

## 保留・スコープ外(参考)

- FreeBSD実機での動作確認(フェーズ7参照。設計上の考慮のみに留める)
- Android等他プラットフォームへの展開(今回のスコープ外)
- POSIX regex.h直接呼び出し、独自wcwidth実装、catgets形式ローカライズ(いずれも不採用済み)
