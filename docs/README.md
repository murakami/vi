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

## コマンド仕様(ユーザー視点)

- [command-spec-ex.md](command-spec-ex.md) — `ex.1`/`TODO`/`Changes`より抽出した、`:`コマンド
  約60種・アドレス指定構文・`:set`オプション約35種・起動オプション・正規表現/置換パターン構文・
  コマンド挙動に影響するChanges上の変更点の一覧。`ex-mode.md`が実装側、こちらが仕様側。
- [command-spec-vi.md](command-spec-vi.md) — `vi.1`より抽出した、visualモードの全コマンド
  (モーション・オペレータ・挿入・検索/置換・マーク/レジスタ・繰り返し/アンドゥ・画面制御・
  モード切替・マクロ等)の一覧。`vi-mode.md`が実装側、こちらが仕様側。
- [command-spec-checklist.md](command-spec-checklist.md) — 上記2文書を元にしたSwift版の
  コマンド互換性チェックリスト(ゴールデンテスト計画のドラフト)。仕様上とくに間違えやすい点、
  Changesベースで「意図的に踏襲する/しない」と判断済みの挙動を個別項目として整理。

## Swift版再実装の検討

- [swift-vi-research-notes.md](swift-vi-research-notes.md) — 上記の調査内容を踏まえ、
  Swiftによる再実装を検討した際の技術方針まとめ(対象プラットフォーム、文字コード方針、
  正規表現、ローカライズ、パッケージ構成案など)。
- [swift-vi-implementation-plan.md](swift-vi-implementation-plan.md) — 実装の進捗管理用
  チェックリスト。フェーズ0(準備)からフェーズ7(検証・仕上げ)まで、`- [ ]`のチェック
  ボックスで進捗を記録する。

## 調査時の注意点・既知の落とし穴

各文書内に記載されている、コードを読んで気づいた注意点の抜粋:

- リカバリ機構の`PRESERVEDIR`はMakefile側の設定とプログラム内のハードコードされた
  パスとの間に不一致がある可能性がある(詳細は`terminal-and-support.md`参照)。
- `libuxre`の照合順序(コレーション)関連コードは`stubs.c`によって実質的に無効化されて
  おり、`colldata.h`のデータが実際には使われていない(詳細は`regex.md`参照)。
- 正規表現エンジンは`UXRE`マクロの有無で全く別の実装(ex独自の簡易REか、libuxreの
  フル実装か)に切り替わる。ビルド設定を確認せずにどちらのコードパスかを断定しない
  こと(詳細は`regex.md`および`CLAUDE.md`参照)。
