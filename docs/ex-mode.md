# ex モードのコア

対象範囲: `ex.c`, `ex_addr.c`, `ex_cmds.c`, `ex_cmds2.c`, `ex_cmdsub.c`, `ex_get.c`, `ex_put.c`,
`ex_set.c`, `ex_subr.c`, `ex_tagio.c`, `ex_temp.c`, `ex_io.c`, `ex_unix.c`, および
`ex.h`/`ex_vars.h`/`ex_temp.h`/`ex_argv.h`。

コマンド仕様(ユーザー視点の一覧)は[command-spec-ex.md](command-spec-ex.md)を参照。本文書は
その仕様を実現する実装アーキテクチャ側の詳細。

## 1. 起動シーケンスと ex/vi/view/edit の分岐

`main()` (ex.c) がすべての起点。ここでの分岐は「その場で処理して終わり」ではなく、
**グローバル変数 `initev`/`globp` に文字列を仕込んで `commands()` のコマンド入力として
後から解釈させる**という間接的な仕組みになっている点が肝。

- `av[0]`(実行ファイル名)に含まれる文字で人格を決定する: `v` を含めば vi、`w` を含めば view
  (`value(READONLY)=1`)、`d` を含めば edit (`SHOWMODE`/`REPORT`等を変更)。実体は全て同じ
  バイナリ `ex`。
- コマンドライン引数 (`-t tag`, `-r`, `+cmd` 等) はここで解析され、`tflag`/`recov`/`firstpat`
  といったグローバルにためられる。
- 起動時に読む設定ファイルは `EXINIT` 環境変数 → `$HOME/.exrc` → カレントの `.exrc` の順
  (`iownit()` で所有者/パーミッションチェック)。読み込みは `commands(1,1)` または `source()`
  経由で、**実際には通常の ex コマンド入力として** 処理される(専用のパーサは無い)。
- `-t`/`-r`/ファイル引数がある場合は `globp` に `"tag"`, `"recover"`, `"next"` という文字列
  コマンド名をセットし、vi 起動でなければ即座に `commands(1,1)` を呼んで実行、vi 起動なら
  `initev` に保存して visual 側 (`ex_vmain.c`) の初期化時に実行させる。
- 最終的に `ivis` (vi として起動) なら `globp="visual"` として `commands(1,1)` を呼び、
  `case 'v':`(下記コマンドディスパッチ)経由で visual モード (`ex_v.c` 以下)に入る。
- 通常の ex 実行はここから素通しで末尾の `commands(0, 0)` に落ち、これがメインループになる
  (vi 側から `Q` や `^\` で抜けた場合もここに合流する)。

## 2. `:` コマンドのディスパッチ ─ テーブルではなく巨大 switch + 前方一致

`commands()` (ex_cmds.c) がメインループ。**コマンドテーブルは存在しない。** 1文字目を
`switch (c)` した巨大な switch 文 (`ex_cmds.c` 内、`'a'`〜`'!'`まで約35ケース)で、各 case が
そのままコマンドの実装(または `ex_cmdsub.c`/`ex_re.c` 等の実装関数への薄いラッパー)になっている。

コマンド名のマッチングは `tail()`/`tailprim()` (ex_cmds2.c:527,543) が担う:

- `tail("append")` のように「今読んでいる入力が `append` という単語の前方一致か」を検証する。
  一致すれば読み進め、後に続く文字が英字ならエラー(`"Not an editor command"`)。
- `d`(delete)に対する `dl`/`dp`、`s`(substitute)に対する `sg`/`sc`/`sr` のように、ed 由来の
  「コマンド名+フラグ文字」が単語境界と誤認されないよう `tailprim()` 内で特別扱いされている
  (ex_cmds2.c:564-567)。
- `tailspec()` (ex_cmds2.c:514) は `=`, `!`, `<`, `>` のような非アルファベットコマンドの
  Command 名をその場で作る。

`commands()` の1ループの流れ:
1. 直前コマンドの後処理 (autoprint/pflag による自動 print、プロンプト表示)。
2. `address(0)` (ex_addr.c) を **カンマ区切りで繰り返し呼んで** `addr1`/`addr2` を確定
   (`;` は評価済みアドレスを `dot` に反映してから継続、という ed 由来の挙動)。
3. `%` は `1,$` の糖衣構文としてその場で展開。
4. `inopen`(open/visual からの `:` エスケープ)中は使えるコマンド集合を絞る。
5. `switch (c)` で command 文字ごとに分岐し、各ケース内で `setdot()`/`setCNL()`/`setnoaddr()`
   等 (ex_addr.c) によりアドレス使用ルールを検証してから実処理へ。

## 3. 行アドレス解析 (`ex_addr.c`)

中心は `address()` (ex_addr.c:217)。1文字ずつ読みながら以下をサポートする再帰的でない
ステートマシン:
- 数値 (`123`)、`.`(dot)、`$`(dol=最終行)、`'x`(マーク、`markreg()`+`getmark()`)
- `+`/`-`/`^` によるオフセット (`+3`, `-`)
- `/pat/`, `?pat?`, `\/`, `\?` による正規表現検索(`compile()`+`execute()` は `ex_re.c` 側。
  `WRAPSCAN` オプションでラップアラウンドの可否を制御)
- 検索パターンを `in_line` 引数越しに渡すことで、`s///` の中で使われる「現在行内での複数マッチ」
  的な特殊系(`doques:` ラベル)も同じ関数で処理している

アドレス確定後のポリシー関数は用途別に分かれる:
- `setdot()`/`setdot1()`: デフォルトアドレスは `dot`。`bigmove`(検索やマーク等 "非局所的" な
  dot 移動があったか)フラグが立っていれば `markDOT()` で `''` (前回位置マーク)を更新する。
- `setcount()`: `delete 5` のような「アドレス + カウント」構文をサポート。
- `setall()`: `:w` などバッファ全体がデフォルトになるコマンド用。
- `setnoaddr()`: アドレス指定を許さないコマンド用(指定されていたらエラー)。

## 4. 編集バッファの内部データ構造 (`ex_temp.c` / `ex_temp.h`)

**ex のバッファは「行ポインタの配列」+「実テキストを格納する一時ファイル」の2層構造**であり、
`ed` と全く同じ設計を踏襲している(ヘッダコメントに明記)。

- 行そのものは core 上の `line` 型(`typedef short/int line`)の配列要素として表現される。
  この値の実体は一時ファイル内のバイトオフセットを詰め込んだもの(`line *dot`, `*dol`, `*zero`
  等はこの配列への **ポインタ**)。`ex_temp.h` 冒頭のコメントいわく「16bit (VMUNIX では32bit)
  に packing され、最下位ビットだけ global コマンド用に予約、残りビットが temp file の
  ブロック番号+オフセットを表す」。
- なぜ一時ファイルベースか: メモリに全行を保持せず、大きなファイルも編集できるようにするため。
  `getline(tl)` (ex_temp.c:197) が `tl` の指す一時ファイル位置から `linebuf` へ1行読み込み、
  `putline()` (ex_temp.c:214) が `linebuf` の内容を一時ファイル末尾(`tline`、常に追記)に書き、
  新しい `tl` を返す。**書き込みは常に追記であり、上書き・GC はしない**(コメントに明記: undo や
  マークの整合性のため、ガベージコレクションは複雑すぎるとして意図的に省略)。つまり編集する
  ほど一時ファイルは肥大化し続ける。
- ブロックキャッシュは3枚: 入力用 `ibuff`/`ibuff2` (2-way LRU, `hitin2`で選択)と出力用 `obuff`
  (常に一時ファイル末尾ブロック)。`getblock()` (ex_temp.c:244) がこの3枚のどれかを指す
  ポインタを返し、`blkio()` (ex_temp.c:299) が実際の `lseek`+`read`/`write` を行う。
- 先頭ブロックは crash recovery 用の `struct header H` (ex_temp.h:179、`Time`/`Uid`/`Flines`/
  `Savedfile`/`Blocks[]`)。`Blocks[]` は行ポインタ配列(core 上の `zero..dol`)自体を一時ファイルへ
  スナップショットしたブロック位置のインデックスで、`synctmp()`/`TSYNC()` (ex_temp.c:348,412)
  がこれを書き出す。クラッシュ時は `expreserve`(後述)がこのヘッダ経由で復元する。
- 名前付きバッファ(レジスタ、a-z と 0-9 の36本)は**編集バッファとは別の一時ファイル**
  (`rfile`/`rfname`)に、独自のブロック割り当てビットマップ `rused[]` (ex_temp.c:505 `REGblk()`)
  を使って保持される。`mapreg(c)` (ex_temp.c:531) がレジスタ文字を `strregs[]` の添字に変換し、
  `putreg()`/`YANKreg()`/`KILLreg()` がレジスタ内容の出し入れを行う。ヤンク/デリート用の無名
  レジスタとは別に、明示的な `"a` 等の指定がこの経路を通る。

## 5. `ex.h`/`ex_vars.h` の主要グローバル

- `line *zero, *one, *dot, *dol, *truedol, *unddol` (ex.h:492-499): 現在のバッファを指す
  行ポインタ群。`zero` は1行目の手前(空)、`dol` が最終行、`dot` がカーソル行。`unddol`/`undap1`/
  `undap2`/`undadot` (ex.h:517-520) は単段 undo 用のスナップショット境界。
- `fendcore`/`endcore` (ex.h:382,387): 行ポインタ配列自体を格納する core 領域の境界。`main()`
  で `sbrk(0)` により確保開始位置を決める(`ex_temp.c` の一時ファイルとは別物で、こちらは
  「行番号→一時ファイル位置」の配列を置く場所)。
- `addr1`/`addr2` (ex.h): 現在処理中コマンドの解析済みアドレス範囲。`ex_addr.c` が書き込み、
  `ex_cmds.c` の各 case が読む、という形でモジュール間の受け渡しに使われる。
- `struct option options[NOPTS+1]` (ex.h:294, 実体は ex_data.c): `:set` オプションテーブル。
  `value(NAME)`/`svalue(NAME)` マクロ (ex.h:308-309) で `options[NAME].ovalue`/`osvalue` に
  展開される。`NAME` は `ex_vars.h` で定義される整数定数(次項参照)。
- `Command` (ex.h:377): 現在実行中のコマンド名文字列。エラーメッセージ等で使用。
- `linebuf[LBSIZE]` (ex.h:406): 現在処理中の1行のワーキングバッファ。`getline()`/`putline()`
  はここを経由してテキストの出し入れをする。
- `names['z'-'a'+2]` (ex.h:408): マーク `a`-`z` および `''`(前回位置)用の行ポインタ配列。

## 6. `:set` オプションの追加方法とビルド時生成の仕組み

`:set` オプションは以下の3ファイルが連動する:

1. `ex_data.c`: `struct option options[NOPTS+1]` の実体(名前, 省略形, 型, デフォルト値等)を
   **アルファベット順**に列挙している配列。新しいオプションを足すときはここに正しい位置で
   1行追加する。
2. `makeoptions`(シェルスクリプト): `ex_data.c` を前処理(`cc -E`)した上で、**既にビルド済みの
   `ex` 自身を ex スクリプトとして起動し** (`ex -s /tmp/...`)、配列の各要素に `nl` で行番号を
   振って `#define ONAME 番号` という形の `ex_vars.h` を生成する(makeoptions:84-119)。つまり
   ビルドは自己ホスト的: 1つ前にビルドされた(あるいはシステムの) `ex` を使って次のオプション
   テーブルを生成する。
3. `ex_vars.h`: 上記で生成される、`value(NAME)` マクロが展開する添字定数の定義ファイル。
   `Makefile` のルール `ex_vars.h: ex_data.c` (Makefile:280) が `ex_data.c` 変更時に
   `makeoptions` を自動実行する。

`ex_set.c` はこのテーブルを引いて `:set name=val` / `:set name` / `:set noname` の構文解析と
値の反映のみを行う(オプションの意味自体は各サブシステムが `value()`/`svalue()` を参照して
実装している)。

## 7. 周辺機能

### シェルエスケープ (`:!`, `!!`, フィルタ)

`unix0()` (ex_unix.c:96) がコマンド行のパース担当。ここで歴史的な `ex`/`csh` 由来の展開が
行われる: `%` → 現在編集中ファイル名 (`savedfile`)、`#` → 直前の別名ファイル (`altfile`)、
`!` → 直前のシェルコマンド全体 (`uxb` に保存済み)の再展開、`\` はこれらのエスケープ。
実行本体は `unixex()` (ex_unix.c:217) で、pipe + fork + `execl(shell, ...)` によって
子プロセスでコマンドを実行し、`filter()` (ex_unix.c:323) は範囲コマンド(`:1,5!sort`)時に
バッファの一部を子プロセスの標準入出力に繋ぎ直す。

### タグジャンプ (`:tag`)

`ex_tagio.c` は伝統的な `tags` ファイル(ctags 形式: `名前\tファイル\t検索パターンまたは行番号`)
を素朴に `topen()`/`tgets()` (ex_tagio.c:99,136) で逐次読み込んで線形探索する実装。バイナリ
検索用のソート済みインデックス等は無い。

### crash recovery (`:preserve`, ハングアップ時の自動保存, `-r` / `exrecover`)

`preserve()` (ex_subr.c:1013) は一時ファイルを `synctmp()` で確定させた後 `execl(EXPRESERVE,
"expreserve", ...)` で別バイナリ `expreserve` (expreserve.c) を起動し、`PRESERVEDIR`
(`/var/preserve`) 配下へ一時ファイルをコピー・登録させる。`onhup()` (ex_subr.c:952) が
`SIGHUP`/`SIGTERM` 受信時にこれを呼ぶため、端末切断や `kill` でも変更内容が失われにくい。
復元側は `ex -r` (main() 内 `recov` 分岐) または `recover()` (ex_unix.c:386) が別バイナリ
`exrecover` (exrecover.c) を fork/exec し、`ex_temp.h` の `struct header` に基づいて一時
ファイルの中身から行ポインタ配列を再構築する。`expreserve`/`exrecover` は `ex` 本体と
別プロセスだが、`struct header` のレイアウトを共有している(ex_temp.h:178 のコメント
「This definition also appears in expreserve.c... beware」が示す通り、手動同期が必要な
デュープした定義)。
