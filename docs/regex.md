# 正規表現エンジン (ex_re.c / libuxre)

コマンド仕様(ユーザー視点の正規表現・置換パターン構文一覧)は[command-spec-ex.md](command-spec-ex.md)
§5を参照。本文書はその仕様を実現する実装アーキテクチャ側の詳細。

exの正規表現機能は2層構造になっている。

1. **`ex_re.c`/`ex_re.h`** — ex独自の「正規表現ラッパー層」。`:s///`(置換)や`/`・`?`(検索)、`:g`(global)コマンドから呼ばれる高レベルAPI(`compile()`, `execute()`, `substitute()`, `dosub()`など)を提供する。
2. **`libuxre/`** — POSIX準拠の正規表現ライブラリ本体(Caldera "UNIX(R) Regular Expression Library" 由来、`regcomp()`/`regexec()`という標準POSIX API形状を持つ独立ライブラリ)。`ex_re.c`はビルド時の`UXRE`マクロの有無でこれを使うか、使わないかが分岐する。

## 1. `UXRE`マクロによるコードパスの分岐 (ex_re.c)

`ex_re.c`はファイル全体が`#ifdef UXRE` / `#else` / `#endif`で二分されている。

- **`UXRE`定義時**(`Makefile`の`REINC`/`RELIB`/`RETGT`が有効、デフォルト): `<regex.h>`(libuxreのヘッダ)をインクルードし、`braslist`/`braelist`/`loc1`/`loc2`をここで実体定義する。`compile1()`は`regcomp()`を呼び、`execute()`は`regexec()`を呼ぶだけの薄いラッパーになる(1242行目以降の`execute()`参照)。
- **`UXRE`未定義時**: `regexp.h`という別の古典的regexpライブラリ(V7/ed系譜のバックトラッキング実装、ex独自にヘッダマクロ`INIT`/`GETC`/`PEEKC`/`UNGETC`/`RETURN`/`ERROR`を定義してインクルードする一種のテンプレート手法)を使う。この場合、大文字小文字を無視した比較のために`loconv()`という独自の小文字化関数(マルチバイト対応)を用意し、パターン文字列そのものを事前に小文字化してから`_compile()`にかける、という力技でIGNORECASEを実現している。

どちらのパスでも上位の`compile()`(1018行目)・`compile1()`(917行目)・`execute()`という関数名は共通で、呼び出し元(`ex_cmdsub.c`など)からは実装の違いを意識しなくてよいようになっている。

## 2. libuxreの全体アーキテクチャ: parse → NFA/DFA → exec

`libuxre/re.h`が全体の型定義のハブになっている。データフローは次の通り:

```
パターン文字列
   │  libuxre_regparse()  (regparse.c)
   ▼
Tree (再帰下降パーサが構築する構文木。ROP_OR/ROP_CAT/ROP_STAR/ROP_BKT等のノード)
   │
   ├─ libuxre_regnfacomp()  (regnfa.c)  → Nfa構造体 (ep->re_nfa)
   │
   └─ libuxre_regdfacomp()  (regdfa.c)  → Dfa構造体 (ep->re_dfa)
```

この分岐は`regcomp()` (regcomp.c:33) の中で行われる。コメント(regcomp.c:45-53)にある通り、ビルドする自動機は次の3パターン:

1. パターンが構造的にNFAを要求する場合(後方参照`\1`、`\<`/`\>`の単語境界、`{n,m}`の巨大な繰り返し(`BRACE_DFAMAX`超え)など。regparse.cの字句解析`lex()`があちこちで`lxp->flags |= REG_NFA`をセットして印を付ける)は、NFAのみ構築。
2. `REG_NOSUB`/`REG_ONESUB`指定時(サブマッチ位置が要らない、単に成否だけ知りたい場合)かつ(1)でなければ、DFAのみ構築。
3. それ以外は両方構築しておき、実行時に選ぶ。

実行側の分岐は`regexec()` (regexec.c:33) にある: `nmatch <= 1`(呼び出し側がサブマッチ位置を要求していない)かつDFAが構築済みならDFAで実行(`libuxre_regdfaexec`、高速)、そうでなければNFAで実行(`libuxre_regnfaexec`、`\(...\)`のキャプチャ位置が取れる)。

exから見ると、これは`ex_re.c`の`execute()` (UXRE版、1244行目) の`nsub = (re.Re_ident == subre.Re_ident ? NBRA : 0)`という判定に直結している。**「今回の正規表現が直前の`:s///`のlhsと同一かどうか」でサブマッチが必要かを決めている** — つまり通常の`/pattern/`検索やglobalの条件式は`nmatch<=1`相当でDFA(高速)、置換コマンドの`&`や`\1`置換のためにサブマッチが必要なときだけNFA、という使い分けがexとlibuxreの境界をまたいで実現されている。

- `regnfa.c`はバックトラック型の明示スタック(`Stack`構造体、regnfa.c内)を使うNFAで、`Graph`ノード(構文木を辿るリンクリスト風表現)上を動く。後方参照や`{n,m}`の巨大反復など、DFAで表現できない機能はここでしか実行できない。
- `regdfa.c`は構文木の各葉(leaf)に位置番号(`left.pos = ep->re_dfa->nposn++`、regdfa.c内`copy()`)を振る、いわゆるGlushkov/Berry-Sethi型の位置オートマトン構成法で、`ROP_BKT`(ブラケット式)は`ROP_BKTCOPY`として複製されつつ元の`Bracket`構造体を共有する。サブマッチ位置が取れない代わりに、後方参照のない大半のパターンに対して高速な決定性実行ができる。

## 3. ブラケット式・照合順序 (bracket.c, _collelem.c, _collmult.c, colldata.h)

`[...]`のコンパイルと実行は`bracket.c`の2つのエントリポイントに集約されている。

- `libuxre_bktmbcomp()` (bracket.c:426) — `[a-z]`, `[:alpha:]`, `[.ch.]`, `[=e=]`などブラケット式全体をパースし、`Bracket`構造体(`libuxre/re.h`で定義。`byte[]`ビットマップ、`wide[]`広域文字配列、`type[]`文字クラス配列、`quiv[]`同値クラス配列を持つ)にコンパイルする。
- `libuxre_bktmbexec()` (bracket.c:687) — 1文字(`wc`)がそのBracketにマッチするかどうかを判定する。判定順序は「文字クラス(`iswctype`) → 単純range(`byte[]`/`wide[]`) → 同値クラス(`quiv[]`) → 否定(`BKT_NEGATED`)時の例外処理」という優先順位で、POSIXの`[B-z]`が大文字小文字両方にマッチする、といった仕様(`NOTES`ファイルに明記されたバグ修正)もここで扱われる。

範囲判定や文字クラス判定の下請けとして`place()`(bracket.c:126)と`mcce()`(bracket.c:203, "multi-character collating element"の略)があり、`mcce()`は**コンパイル時と実行時の両方**で呼ばれる共用ロジックである点が特徴的(コメント177-190行目): コンパイル時は「消費すべきバイト数が確定している」動作、実行時は「マッチしうる最長の照合要素を探す」動作、という違いをフラグ(`compile_time`引数)で切り替える。

照合順序(collation)のデータは`colldata.h`で定義される`CollHead`/`CollElem`/`CollMult`/`CollSubn`という一連のオンディスク構造体で表現される設計になっており(コメントに詳細なバイナリレイアウト説明あり)、本来はロケールごとの照合順序ファイルをmmapして使う想定である。しかし`stubs.c`の`libuxre_lc_collate()`が常に`CHF_ENCODED`(単純なエンコード値順、つまり照合順序を無視してバイト値/コードポイント順で比較)を返すスタブ実装になっているため、**実際にはLC_COLLATEに基づく本格的な照合順序は機能しない**。これはリポジトリルートの`TODO`に書かれている「LC_COLLATEロケールは完全に無視される」という記述と一致する既知の制限である。`_collelem.c`(`libuxre_collelem()`)と`_collmult.c`(`libuxre_collmult()`)はこのCollElem/CollMultテーブルを引く補助関数で、スタブ状態でも「1文字1エントリのCHF_ENCODED」経路として動作する。

## 4. マルチバイト対応 (wcharm.h)

`wcharm.h`は「広域文字ロケール情報のスタブ」的な薄いヘッダで、次の抽象を提供する。

- `w_type`(`int`のtypedef): 正規表現エンジン内部で文字を表す型。マイナス値はROP_*演算子の符号化に使われるため(`libuxre/re.h`の`MAKE_ROP`マクロ参照)、`w_type`が有効な文字値をすべて正の範囲で表現できることが前提になっている。
- `ISONEBYTE(ch)`: `(ch & 0200) == 0 || mb_cur_max == 1` — そのバイトが1バイト文字(ASCII相当)か、あるいはそもそもマルチバイトロケールでないかを判定するマクロ。`mb_cur_max`(exの`ex.h`側で管理されるグローバル)が1固定ならマルチバイト処理を丸ごとスキップできるようになっている。
- `to_lower`/`to_upper`: `mb_cur_max > 1`なら`towlower`/`towupper`(wctype.h)、そうでなければ`tolower`/`toupper`を使う切り替えマクロ。
- `libuxre_mb2wc()` (stubs.c:42): `mbtowc()`を呼んで1マルチバイト文字を`w_type`にデコードする実装。`regparse.c`の`lex()`内、`ISONEBYTE()`で単純処理できないと判定されたときに呼ばれる(regparse.c:668-677)。

つまりマルチバイト対応は「1バイトで済む場合は高速パス、そうでない場合だけ`mbtowc`/`wctomb`/`towlower`等の広域文字APIを呼ぶ」という2段構えで、正規表現エンジンの各所(`bracket.c`のクラス判定、`regparse.c`の字句解析)に同じパターンで埋め込まれている。

## 5. onefile.c / stubs.c の役割

- **`onefile.c`**: `_collelem.c`, `_collmult.c`, `stubs.c`, `bracket.c`, `regdfa.c`, `regnfa.c`, `regparse.c`, `regcomp.c`, `regexec.c`を`#include`で1つの翻訳単位にまとめ、`LIBUXRE_STATIC`を`static`に再定義するアマルガメーション(単一ファイル化)ビルド用のソースである。**ただし`libuxre/Makefile`はこれを使っておらず**、各`.c`を個別にコンパイルして`libuxre.a`にアーカイブする通常のビルドを行っている。`onefile.c`はCalderaの元プロジェクト(`osutils`)由来で、他プロジェクトへ本ライブラリを1ファイルとして静的リンクなしで埋め込みたい場合の選択肢として残されているものと考えられる。
- **`stubs.c`**: 「libcのRE関連コードを完成させるために必要なスタブ関数群」(ファイル冒頭コメント)。具体的には照合順序の`libuxre_lc_collate()`(常にCHF_ENCODED固定 = 照合順序機能を無効化)と、マルチバイトデコードの`libuxre_mb2wc()`(こちらは本実装であり、スタブではなく実処理)の2つを提供する。ファイル末尾にはSCCS ID文字列を1箇所にまとめたコメントブロックがあり、ビルド全体のバージョン管理用途も兼ねている。

## 6. exコマンドからlibuxreへの呼び出し経路

- **検索** (`/pattern`, `?pattern`): `ex_re.c`の`compile()` (1018行目) がexコマンドラインからパターン文字列を読み取り、`re.Patbuf`に正規化して格納した上で`compile1()` → `regcomp()`を呼ぶ。マッチ自体は(vi visualモードの検索コマンド側から、あるいは`ex_addr.c`のアドレス解析`/re/`から)`execute(0, addr)`を介して行われ、成功すると`loc1`/`loc2`(マッチ区間の開始・終了ポインタ、`linebuf`内)がセットされる。
- **置換** (`:s/re/rhs/flags`): `compsub()` (386行目) がlhs(検索パターン)をコンパイルし、`comprhs()` (457行目) がrhs(置換文字列、`&`や`\1`等のエスケープを含む)を`rhsbuf`にコンパイルする。`substitute()` (341行目) が対象行範囲をループし、各行で`dosubcon()` → `execute()` → `confirmed()`(`c`フラグ時の確認プロンプト) → `dosub()` (666行目、実際に`genbuf`へ書き戻す本体) という流れで置換を適用する。`dosub()`内で`braslist[c-'1']`/`braelist[c-'1']`(サブマッチの開始・終了)を使って`\1`等を展開しており、ここがまさに前述の「置換はNFA必須」の理由になっている箇所である。
- **global** (`:g/re/cmd`): `global()` (181行目) が全行に対して`execute()`でマッチ判定を行い、マッチした行に印(最下位ビット)を付けた上で、`commands()`(ex_cmds.c側のディスパッチループ)を印の付いた行ごとに再実行する仕組み。
