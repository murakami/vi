# 端末層・リカバリ機構・メモリアロケータ

`ex`/`vi`本体(`ex.c`ほか)を支える3つの補助サブシステムについてまとめる。いずれも
`ex`のコアロジックとは疎結合で、それぞれ独立したファイル群として存在する。

## 端末層 (libterm/)

`libterm/`は2.11BSD由来のtermcapライブラリをほぼそのまま移植したもので、`ex`が
raw tty制御や画面描画に必要な「端末の能力(capability)」を得るための唯一の情報源に
なりうる。Makefile変数`TERMLIB`が`termlib`のときにこのライブラリがビルド・リンク
され(`libterm/Makefile`が`libtermlib.a`を生成)、`TERMLIB=curses`/`ncurses`を選ぶと
このディレクトリは使われず、代わりにシステムのcurses/ncursesが持つterminfoベースの
互換API(`tgetent`等の名前を持つラッパー)がリンクされる。ex側のコード(`ex_tty.c`など)
は`libterm.h`が宣言する5関数だけを見ており、どちらの実装を使うかはリンク時に決まる。

### capabilityの取得: `tgetent` (termcap.c)

`tgetent(bp, name)`が端末データベースの探索窓口。動作は環境変数`TERMCAP`の中身で
分岐する:

- `TERMCAP`が`/`で始まる → その値をファイルパスとして`/etc/termcap`の代わりに読む。
- `TERMCAP`がエントリ文字列そのもの(改行が除去済み)を含む → ファイルI/Oを一切せず、
  `tnamatch()`で名前が一致すればその文字列をそのままバッファにコピーする。
- どちらでもない、または`TERMCAP`が未設定 → `/etc/termcap`(`E_TERMCAP`)を素の`read()`
  ループで1行ずつ読み、`tnamatch()`で`name`と一致するエントリを探す。

エントリが見つかった後は`tnchktc()`が呼ばれ、`tc=xxx`という「他のエントリを継承する」
指定を再帰的に展開する(`MAXHOP`=32段までの循環検出付き)。ここまでで`tbuf`という
static変数にエントリ全体の生文字列がセットされ、以降の`tgetnum`/`tgetflag`/`tgetstr`は
すべてこの`tbuf`を線形スキャンするだけの単純な実装になっている(データベースはコンパイル
されず毎回スキャンする、という設計上の既知の欠点がコメントに明記されている)。

- `tgetnum(id)`: `li#80`のような`#`区切りの数値capability。
- `tgetflag(id)`: `bs`のような真偽値capability(存在すれば1)。
- `tgetstr(id, area)`: `cl=...`のような文字列capability。`tdecode()`が`^X`や`\E`などの
  エスケープ表記をデコードし、呼び出し元が渡した`area`バッファに書き出しながらポインタを
  進める(呼び出し側はバッファオーバーフローをチェックしない、と明記されている)。

### カーソル移動と出力: `tgoto.c` / `tputs.c`

`tgoto(CM, destcol, destline)`はcapability文字列`cm`(カーソル位置指定の
printf風テンプレート、`%d`, `%2`, `%i`, `%>xy`など独自の書式指定子を解釈)を実際の
エスケープシーケンスに展開する。行/列の入れ替え(`%r`)や1始まりインデックスへの変換
(`%i`)、null/EOT/改行になってしまう値を避けるための特殊ケース処理(`UP`/`BC`変数を
使ったバックスペース・カーソル上移動での代替)など、歴史的な端末の癖を吸収するための
ロジックが集中している。

`tputs(cp, affcnt, outc)`は生成された文字列を1文字ずつ`outc`コールバックへ渡して
出力するだけでなく、capability文字列の先頭にある`<数値>[.<小数点1桁>][*]`という
パディング指定を解釈し、`ospeed`(回線速度)に応じたNUL文字(`PC`)の遅延送出を行う。
低速端末の画面描画待ち時間を作るための機構で、現代の端末エミュレータでは実質的に
無効(`ospeed`が範囲外なら即座に`return`)。

## リカバリ機構 (expreserve / exrecover)

`ex`はクラッシュやハングアップからの復旧のために、編集バッファを別プロセスへ退避する
仕組みを持つ。中心となるのは`ex_temp.c`が管理する一時ファイル(バッファの実体、
`docs/ex-mode.md`参照)そのものであり、`expreserve`/`exrecover`はその一時ファイルを
読み書きするだけの小さな独立バイナリである。

### 保存: `preserve()` (ex_subr.c) → `expreserve` (expreserve.c)

`ex`本体の`preserve()`(ex_subr.c:1012)が退避のトリガ。`synctmp()`で一時ファイルを
フラッシュした後`fork()`し、子プロセスの標準入力を一時ファイル(`tfile`)に付け替えて
`execl(EXPRESERVE, "expreserve", (char *)0)`を実行する。`EXPRESERVE`は`ex_tune.h`で
`"/usr/sbin/expreserve"`と定義されるコンパイル時定数で、`Makefile`の`RECOVER`変数
(`-DEXPRESERVE=...`)経由で上書きできる。

`expreserve`(引数なしで呼ばれた場合)は標準入力から`struct header H`(退避対象ファイルの
最終更新時刻・uid・行数・元ファイル名・一時ファイル内の行ポインタが格納されたブロック
番号配列)を読み込み、`copyout()`でヘッダの整合性(行数が非負か、`H.Blocks[0]`/`[1]`が
期待値`HBLKS`/`HBLKS+1`か、uidが一致するか)を検証したうえで、`/var/preserve`配下に
`Exa\`XXXXXXXXXX`という名前(`mkdigits()`でPID、`mknext()`でアルファベット部分を
一意になるまでインクリメント)のファイルを作成し中身を丸ごとコピーする。成功すると
`notify()`が`popen("/bin/mail ...")`経由で所有者にメールを送り、`ex -r <file>`で
復元できる旨を知らせる。

なお`root`で`expreserve`に何らかの引数を1つ渡して呼び出すと、`argc==1`分岐ではなく
「`/var/tmp`(`TMP`)以下の`Ex`で始まる全ファイルを一括退避する」バッチモードになる
(システムシャットダウン時にinitスクリプトなどから呼ばれることを想定した経路。
`ex`本体からはこの経路は使われない)。

**注意点**: `expreserve.c`内の保存先は`pattern[] = "/var/preserve/Exa\`XXXXXXXXXX"`と
ソース中に直書きされており、`Makefile`の`PRESERVEDIR`変数(デフォルトも`/var/preserve`
で一致するが)を変更しても自動的には反映されない。`PRESERVEDIR`はインストール時に
そのディレクトリを作成・sticky bit付与するためだけに使われる(`make install`)。
配置先を変える場合は`expreserve.c`の`pattern`と`exrecover.c`の`mydir`を両方手で
書き換える必要がある(`exrecover.c`内のコメントにも「両方変えること」と明記あり)。

### 復元: `recover()` (ex_unix.c) → `exrecover` (exrecover.c) → `ex.c`の初期コマンド

2つの経路がある。

1. `ex -r`(引数ファイルなし): `ex.c`の`main()`(ex.c:544)が直接
   `execl(EXRECOVER, "exrecover", "-r", (char *)0)`を実行し、`exrecover`の
   `listfiles()`が`/var/preserve`と`/var/tmp`(正確には呼び出し時に渡すディレクトリ引数)
   を走査、自分のuidと一致する退避ファイルのヘッダを読んで一覧表示して終了する。
2. `ex -r <filename>`または`:recover`コマンド実行時: `ex_unix.c`の`recover()`が
   パイプを`pipe()`で作成し、`fork()`した子プロセスの標準出力をパイプの書き込み側に
   付け替えたうえで`execl(EXRECOVER, "exrecover", svalue(DIRECTORY), file, (char *)0)`
   を実行する。親(ex本体)はパイプの読み取り側`io`から届く再構成済みファイル内容を
   通常のファイル読み込みと同じ経路で取り込む。

`exrecover`の`main()`は`ex_temp.h`が定義する行番号↔一時ファイルオフセットの変換ロジック
(`getblock()`, `getline()`, `blkio()` — `ex_temp.c`にある実装をほぼそのまま持ち込んだもの)
を使い、`findtmp()`→`searchdir()`→`yeah()`で候補ファイルの中から指定ファイル名・uidが
一致し、かつ最も新しい(`H.Time`が最大)ものを選ぶ。選んだ一時ファイルのブロック配列
(`H.Blocks[]`)を辿って行ポインタ配列を`sbrk()`で確保した領域上に再構築し、
`scrapbad()`でシステムクラッシュ時の書き込み順序の乱れによって生じた「宙に浮いた」
行ポインタ(ファイル末尾を超えて指すもの)を検出して"LOST"扱いにしたうえで、
`putfile()`で標準出力(=exへのパイプ)に書き戻す。読み終えた退避ファイルはその場で
`unlink()`される。

## メモリアロケータ (malloc.c / mapmalloc.c)

### なぜlibcのmalloc()を置き換える必要があるか

`ex`は編集バッファの行ポインタ配列を保持するための領域を、`sbrk()`を直接呼んで
プロセスのデータセグメントを伸張することで確保している(`main()`の`fendcore = (line *)
sbrk(0)`、および`ex_subr.c`の`morelines()`が行う`sbrk(pg * sizeof(line))`ないし
`sbrk(1024 * sizeof(line))` — ex.c:525, ex_subr.c:493,498)。もしlibcの`malloc()`も
同時に`sbrk()`でヒープを伸ばす実装だった場合、`ex`が直接`sbrk()`で伸ばした領域と
libc malloc内部の管理領域が同じアドレス空間を奪い合い、互いのメタデータを破壊しうる。
これを避けるため、`ex`は`malloc()`/`free()`/`realloc()`/`calloc()`をリンク時に
自前の実装で上書きし、`Makefile`の`MALLOC=`変数でどちらを使うか選択する
(README「Prerequisites」にも同旨の記載あり)。

両実装とも中身は同じ「循環ファーストフィット(circular first-fit)」アルゴリズム
(AT&T Unix 7th Editionのmalloc)がベースで、確保領域はブロック単位の連結リストとして
管理され、各ブロックの先頭ワードに「次のブロックへのポインタ」と「使用中/空きを表す
最下位ビット(`BUSY`)」を同居させる。`malloc()`は現在の探索ポインタ`allocp`から
リストを一周探索し、連続する空きブロックを`while(!testbusy(...))`ループでその場で
併合(coalescing)しながら十分なサイズを探す。見つからなければアリーナを拡張して
リトライする。`free()`は該当ブロック直前のポインタワードの`BUSY`ビットを落とすだけ
(LIFO的な再利用を想定した設計、と冒頭コメントに明記)。

- **malloc.c**(`MALLOC=malloc.o`): アリーナの拡張を`poolsbrk()`という自前の
  ラッパー経由で行う。`poolsbrk()`は初回呼び出し時に`sbrk(POOL)`(`POOL`=32768バイト)
  で固定サイズのプールを一括確保し、以降はそのプール内をオフセットで切り売りするだけ
  (`sbrk()`を2回目以降は呼ばない)。プールを使い切ると`error("Memory pool exhausted")`
  で異常終了する。コメントによれば、`ex`自身は内部で`malloc()`をほぼ使わず
  (行バッファは`sbrk()`直接)、`setlocale()`など外部ライブラリ呼び出しのためだけに
  この経路が必要になる、比較的小さなプールで足りるとされている。
- **mapmalloc.c**(`MALLOC=mapmalloc.o`、デフォルト): `sbrk()`を一切使わず、
  `mmap(MAP_ANON|MAP_PRIVATE)`(`MAP_ANON`が無ければ`/dev/zero`をオープンして代用)で
  プールを確保する`struct pool`の連結リストとして実装されている。1つのプールを
  使い切ると`poolblock`を1.5〜2倍に増やしながら新しいプールを`mmap()`し、
  `pool->Next`でリンクして探索を続ける(動的に成長できるため、固定サイズの
  `malloc.c`版よりも一般的には好ましい、とMakefileのコメントに記載)。各割り当て
  ブロックの直後に確保元プールへのポインタ(`p[1].pool`)を埋め込むことで、
  `free()`/`realloc()`がどのプールに属するブロックかを判別できるようにしている。

いずれの実装も`VMUNIX`が定義されているときのみ有効(`#ifdef VMUNIX` … `#endif`で
ファイル全体が囲われている)。

## 補足: printf.c

`printf.c`はVersion 7 C相当の縮小版`printf`/`vprintf`を独自実装したもの
(浮動小数点や高度な書式指定は非対応、`putchar()`経由の出力のみ)。stdioの`printf`が
内部でlibc mallocを使う可能性を避け、かつバイナリサイズを抑えるための歴史的な工夫で、
`ex.h`が宣言する`ex`独自のI/Oバッファリング層と組み合わせて使われる。
