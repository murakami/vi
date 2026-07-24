# ex/vi ソースコード調査ドキュメント

このディレクトリは、伝統的な`ex`/`vi`エディタ移植版(このリポジトリ)のソースコードを
サブシステムごとに詳細調査した結果をまとめたものです。全体的な概要・ビルド方法は
リポジトリルートの`CLAUDE.md`を参照してください。ここではCLAUDE.mdより一段深い、
複数ファイルにまたがる制御フローやデータ構造の詳細を記録しています。

## 文書一覧

- [ex-mode.md](ex-mode.md) — exモードのコア。`main()`の起動シーケンスとex/vi/view/edit
  分岐、`:`コマンドのディスパッチ(巨大switch+前方一致)、`ex_addr.c`の行アドレス解析、
  `ex_temp.c`の編集バッファ内部構造(行ポインタ配列+追記専用一時ファイル)、
  `ex.h`/`ex_vars.h`の主要グローバル、`:set`オプションの追加方法(`ex_data.c`/
  `makeoptions`/`ex_vars.h`の3点連動)、シェルエスケープ・タグジャンプ・crash recovery
  の呼び出し経路。
- [vi-mode.md](vi-mode.md) — viビジュアル(スクリーン編集)モード。exモードからの遷移と
  メインループ構造、モーション+オペレータの2段階デコード(`operate()`, `ex_voper.c`)、
  フル再描画と差分更新の使い分け、`ex_vadj.c`/`ex_vwind.c`によるウィンドウ調整と
  スクロール、raw ttyモードの設定・解除、主要コマンドの実装場所。
- [regex.md](regex.md) — 正規表現エンジン。`ex_re.c`における`UXRE`マクロでのコード
  パス分岐、`libuxre`の parse(`regparse.c`) → NFA(`regnfa.c`)/DFA(`regdfa.c`) → 実行
  (`regexec.c`)という全体アーキテクチャ、ブラケット式・照合順序(`bracket.c`,
  `_collelem.c`, `_collmult.c`, `colldata.h`)、マルチバイト対応(`wcharm.h`)、
  `onefile.c`/`stubs.c`の役割、`:s///`や検索からlibuxreへの呼び出し経路。
- [terminal-and-support.md](terminal-and-support.md) — 端末層(`libterm/`)・リカバリ
  機構・メモリアロケータ。`termcap.c`によるcapability取得、`tgoto.c`/`tputs.c`による
  カーソル移動とパディング出力、`preserve()`→`expreserve`と`recover()`→`exrecover`の
  保存/復元の2経路、`malloc.c`(静的プール)と`mapmalloc.c`(mmapベース)がlibc mallocを
  置き換える理由とアルゴリズム概要。

## 調査時の注意点・既知の落とし穴

各文書内に記載されている、コードを読んで気づいた注意点の抜粋:

- リカバリ機構の`PRESERVEDIR`はMakefile側の設定とプログラム内のハードコードされた
  パスとの間に不一致がある可能性がある(詳細は`terminal-and-support.md`参照)。
- `libuxre`の照合順序(コレーション)関連コードは`stubs.c`によって実質的に無効化されて
  おり、`colldata.h`のデータが実際には使われていない(詳細は`regex.md`参照)。
- 正規表現エンジンは`UXRE`マクロの有無で全く別の実装(ex独自の簡易REか、libuxreの
  フル実装か)に切り替わる。ビルド設定を確認せずにどちらのコードパスかを断定しない
  こと(詳細は`regex.md`および`CLAUDE.md`参照)。
