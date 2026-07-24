# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## これは何か

Gunnar Ritter氏による「ex-vi」ポート版であり、伝統的なAT&T/BSDの`ex`/`vi`ライン・スクリーンエディタです。
ex/vi 3.7（1985/6/7版）と2.11BSDのtermcapライブラリを元に、現代のPOSIX準拠システムへ移植し、
その上にマルチバイト/UTF-8ロケール対応を追加したものです。C89で書かれた単一バイナリで、
`make`以外のビルドシステムは使用していません。テストスイート、CI、パッケージマニフェストは
存在しません — これは歴史的なUnixシステムのソースコードであり、現代的な意味でのアプリケーションではありません。

上流プロジェクト: <http://ex-vi.sourceforge.net>。ビルド/移植に関する詳細は`README`を、
オリジナルのBSDソースからの変更履歴は`Changes`を参照してください。

## ビルド

設定は独立したconfigureスクリプトではなく、`Makefile`（先頭の変数群）と`config.h`
（`TUBELINES`/`TUBECOLS`などのバッファサイズ、`ex_tune.h`のチューニング値）に直接記述されています。
新しいプラットフォームでビルドする前に、以下を確認・編集してください。

- `Makefile`: `TERMLIB`（`termlib`（同梱・デフォルト）、`curses`、`ncurses`のいずれか1つを選択）、
  `MALLOC`（デフォルトは`mapmalloc.o`、`mmap()`によるアノニマスメモリ確保が使えない環境では
  `malloc.o`）、`FEATURES`（マルチバイト対応の`-DMB`、`-DBIT8`など）、`OSTYPE`（現代的なUnixでは
  `-DVMUNIX`）、`INSTALL`/`PREFIX`/`BINDIR`など。
- `REINC`/`RELIB`/`RETGT`: 同梱の`libuxre`（ロケール対応のブラケット表現をサポートするPOSIX正規表現
  ライブラリ）を指定します。ターゲットのコンパイラでマルチバイト対応正規表現ライブラリがビルドできない
  場合は、この3行をコメントアウトしてください。

```sh
make            # libuxre, libterm, ex, exrecover, expreserve をビルド
make install     # ex 本体とシンボリックリンク(edit, vedit, vi, view)、manページをインストール
make clean       # このディレクトリ、libterm/、libuxre/ のオブジェクトファイルをクリーン
```

このリポジトリにはlint/test/formatの類のツールはありません。変更の検証はビルドを通すことと、
`ex`/`vi`のコマンドを手動で実際に動かして確認することで行ってください（代わりに実行できる
自動テストハーネスはありません）。

## アーキテクチャ

エディタ全体は単一の実行ファイル（`ex`）であり、`argv[0]`に応じて`ex`・`vi`・`view`・`edit`
として振る舞いを変えます（`ex.c`の`main()`が`av[0]`を`any('v', ...)` / `any('w', ...)` /
`any('d', ...)`で調べ、人格を決定しています）。エディタのグローバルな状態（バッファポインタ、
モードフラグ、行アドレスのグローバル変数など）は`ex.h`/`ex_vars.h`にあり、ほぼ全ての`.c`ファイル間で
共有されています — モジュールによるカプセル化は存在せず、ほとんどのファイルが同じグローバル状態へ
直接アクセスします。

ソースコードは歴史的なBSD `ex`のソース構成にほぼ沿って、サブシステムごとに分割されています。

- **コマンドライン/exモードのコア**: `ex.c`（エントリポイント、引数解析、トップレベルディスパッチ）、
  `ex_cmds.c`/`ex_cmds2.c`/`ex_cmdsub.c`（`:`コマンドの実装）、`ex_addr.c`（`1,$`のような行/アドレス
  解析）、`ex_get.c`/`ex_put.c`（バッファとファイル間の読み書き）、`ex_set.c`（`:set`オプション、
  一部は`makeoptions`により`ex_vars.h`へ生成される）、`ex_subr.c`（共通サブルーチン）、`ex_tagio.c`
  （`ctags`対応）、`ex_temp.c`（一時ファイルを裏付けとする編集バッファ — 編集中ファイルの実際の
  メモリ上の表現）、`ex_io.c`（端末/ファイルI/Oのグルーコード）、`ex_unix.c`（プロセス/シグナル/
  シェルエスケープの処理）。
- **ビジュアル（vi）モード**: `ex_v.c`、`ex_vmain.c`（visualモードのメインループ）、`ex_vget.c`
  （キー入力）、`ex_vops.c`/`ex_vops2.c`/`ex_vops3.c`（`d`、`c`、`y`などの操作コマンド）、
  `ex_vput.c`（画面出力/再描画）、`ex_vadj.c`（画面/ウィンドウ調整）、`ex_vwind.c`（ウィンドウ
  サイズ処理）、`ex_voper.c`（オペレータのディスパッチ）、`ex_tty.c`（raw tty モードの処理）。
- **正規表現**: `ex_re.c`/`ex_re.h`はex独自の正規表現レイヤー、`libuxre/`は同梱の「Caldera UNIX
  Regular Expression Library」派生版で、POSIXブラケット表現（`[:class:]`、`[.c.]`、`[=c=]`）、
  ロケール照合、マルチバイト対応のマッチングを提供します。静的ライブラリとしてビルドされ、
  `UXRE`定義時に`RELIB`/`REINC`経由でリンクされます。
- **端末機能**: `libterm/`は自己完結型のBSD termcapライブラリ（`termcap.c`、`tgoto.c`、`tputs.c`）
  で、`TERMLIB=termlib`のときに使用されます。代替として`curses`/`ncurses`をリンクする構成も可能です
  （Makefileの端末機能に関するコメントのトレードオフ説明を参照）。
- **リカバリデーモンのペア**: `expreserve.c`（クラッシュ/ハングアップ時に編集バッファを保存）と
  `exrecover.c`（`ex -r`、保存されたバッファを復元）は`ex`とは別の小さなバイナリとしてビルドされ、
  `PRESERVEDIR`（`/var/preserve`）以下のファイルを介して通信します。
- **独自アロケータ**: `ex`はlibcの`malloc()`を独自実装（`malloc.c`または`mapmalloc.c`、Makefileの
  `MALLOC=`で選択）に置き換えています。これは`sbrk()`/`mmap()`を直接管理するためです。`README`に
  記載されている通り必須要件であり、アロケーション関連コードを触る際はlibc mallocのセマンティクスを
  前提にしないでください。
- `printf.c`: 独自のprintf実装（歴史的な移植性の理由により、libcへの依存を避けたもの）。
- `catd/en_US`: `LANGMSG`/`catgets()`対応のためのメッセージカタログデータ（デフォルトでは無効な
  オプトイン機能）。

`ex_proto.h`は`.c`ファイル群に合わせて集約・維持される関数プロトタイプヘッダです。`ex_vars.h`は
ビルド時に`makeoptions`シェルスクリプトによって`ex_data.c`から生成されます（`:set`オプション表）。
新しい`:set`オプションを追加する際は、`ex_set.c`だけでなく`ex_data.c`と`makeoptions`もあわせて
確認してください。

## このコードベースで作業する際の注意点

- これは歴史的なK&R/C89寄りのコードで、多数のグローバル変数と簡潔な歴史的命名規則（1文字/短い
  識別子、随所に見られる`register`、`bool`のtypedefなど）が使われています。その場で近代化するのではなく、
  既存のスタイルに合わせてください。
- 機能フラグはコンパイル時（`-DMB`、`-DBIT8`、`-DLISPCODE`、`-DCHDIR`、`-DFASTTAG`、`-DUCVISUAL`、
  `-DUXRE`、`-DLARGEF`、`-DLANGMSG`、`-DVFORK`）で`Makefile`の`FEATURES`/`OSTYPE`/`LARGEF`変数
  経由で指定されます。あるコードパスが有効かどうかは、どのフラグが有効化されているかを確認してから
  判断してください。
- マルチバイト対応（`-DMB`）には`wcwidth()`/`mbrtowc()`が必要です。無効な場合は非マルチバイトの
  コードパスと非UXREの正規表現エンジンが代わりに使われます。同じファイル内に`#ifdef MB`/
  `#ifdef UXRE`で両方のパスが存在することに注意してください。
- `ex.1`と`vi.1`はユーザー向けのコマンド/オプションに関する正式なドキュメント（troff形式の
  manページ）です。ユーザーに見える挙動（コマンド、`:set`オプション、CLIフラグ）を変更した場合は
  これらも更新してください。
