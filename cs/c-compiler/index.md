# Cコンパイラ

Linux上で動作するx86-64用のCコンパイラを作ったときのメモ。自分が実装するときに考えたことをまとめました。間違っている部分も多くあると思います。特にセルフホストの前処理の部分は自信がありません。

## プログラムの実行

人間が書いたソースコードはプリプロセスを経た後、コンパイラによってアセンブリコードに変換され、更にアセンブラによってオブジェクトコードに変換される。複数のオブジェクトファイルはリンカによって結合され、最終的な実行ファイルが生成される。

プログラムを実行する際、OSのプログラムローダが実行ファイルの内容をメモリ上にコピーする。メモリは以下の4つの領域に分かれている。

- テキスト領域: 命令
- 静的領域: 静的変数
- ヒープ領域: 動的確保される領域
- スタック領域: 静的確保される領域

プログラムを実行するCPUは、レジスタやメモリからデータを読み込み、演算結果を再びレジスタやメモリに書き込む。レジスタはCPU内部に存在するので、メモリに比べてアクセスが速い。CPUはプログラムカウンタというレジスタに現在実行中の命令のアドレスを保持しており、そのアドレスから順次命令を読みだして実行する。

UNIXシステムにおけるオブジェクトファイルと実行ファイルには同じフォーマットが用いられる。過去にはa.outやCOFF(Common Object File Format)などのファイル形式が使われていたが、現在ではELF(Executable and Linkable Format)が広く採用されている。

## C言語

### 歴史

- 1971 4004: ビジコンとIntelが開発した世界初の4ビットマイクロプロセッサ。
- 1974 Intel 8080: 8ビットマイクロプロセッサ。
- 1978 Intel 8086: 16ビットマイクロプロセッサ。
- 1978 _The C Programming Language_(Brian W. Kernighan, Dennis M. Ritchie)
- 1979 BDS-C: Intel 8080/Z80用に開発されたK&R準拠のCコンパイラ
    - Z80: 1976年に発表された8ビットマイクロプロセッサ。Intel 8080の上位互換。
- 1985 Intel 80386(IA-32): 32ビットマイクロプロセッサ。
- 1987 GCC
- 1989 ANSI X3.159-1989(ANSI C, C89)
- 1990 ISO/IEC 9899:1990(C90)
- 1995 ISO/IEC 9899/AMD1:1995(C95)
- 2000 ISO/IEC 9899:1999(C99)
- 2011 ISO/IEC 9899:2011(C11)
- 2018 ISO/IEC 9899:2018(C17, C18)

## x86アセンブリ言語
gccなどの処理系では、アセンブラとして主にGNUアセンブラ(GNU assembler, gas)が用いられており、GNUアセンブラが出力するアセンブリ言語もGNUアセンブラの仕様書で定義されている。アセンブリ言語の文法には一部アーキテクチャに依存する部分がある。

アセンブリ言語は以下の4つからなる。
- 機械語命令(machine instruction): CPUが実行時に実行。
- アセンブラ命令(assembler directive): アセンブラがアセンブル時に実行。実行ファイルに残らない。
- ラベル: ジャンプ先の指定など。
- コメント

GNUアセンブラの場合、アセンブラ命令は全て`.`から始まる。

### AT&T記法とIntel記法
x86系のアセンブリ言語には大きくAT&T記法とIntel記法という二つの文法がある。
- sigil
    - AT&T記法: 即値に$を付け、レジスタ名に%を付ける。
    - Intel記法: 何も付けない。
- オペランドの順番
    - AT&T記法: mnemonic source, destination
    - Intel記法: mnemonic destination, source
- メモリオペランドのサイズ
    - AT&T記法: ニーモニックの末尾に`b`, `w`, `l`, `q`などを付ける。
    - Intel記法: メモリオペランドの前に`byte ptr`, `word ptr`, `dword ptr`, `qword ptr`などの接頭辞を付ける。
- メモリ参照

GNUアセンブラではデフォルトでAT&T記法を用いるが、2.10からは`.intel_syntax`ディレクティブを付けることでIntel記法が使えるようになった。ここではIntel記法を使う。

### メモリ参照
- 間接メモリ参照: [base + index * scale + disp]
    - base, index: レジスタ。
    - scale: 1,2,4,8のいずれか。指定しない場合1。
- RIPアドレッシング: [rip + offset]
    - offset: 数値かシンボル。

QWORD PTR [reg]はregのアドレスから8バイト

### アセンブラ命令
#### セクション指定
- `.text`: テキスト領域
- `.data`: データ領域
- `.rdata`
- `.bss`

#### シンボル情報
- `.global`, `.globl`: シンボルをリンカldに認識させる。

#### パディング
- `.space`: n byteごとに値を格納する。デフォルトは0。`.skip`と同じ。
- `.skip`: n byteごとに値を格納する。デフォルトは0。`.space`と同じ。
- `.zero`: n byteに0を格納。`.skip`のエイリアス。
- `.align`: システムによってn byte境界または2^n byte境界に合わせる。
- `.p2align`: ロケーションカウンタを2^nの倍数になるまで進める。

#### データ配置
- `.byte`: 0個以上の式を1byteごとに配置。
- `.short`: 0個以上の式を2byteごとに配置。
- `.word`: 0個以上の式を配置。サイズはターゲットに依存する。
- `.value`: `.short`と同じ。x86専用([9.16.2 x86 specific Directives (Using as)](https://sourceware.org/binutils/docs/as/i386_002dDirectives.html))。
- `.int`: 0個以上の式を配置。バイトオーダーとサイズはターゲットに依存する。
- `.long`: `.int`と同じ。
- `.quad`: 0個以上の式を8byteごとに配置。

#### 文字列リテラル
- `.string`: 文字列に終端文字を付けて配置。
- `.ascii`: 0個以上の文字列リテラルを連続して配置する。
- `.asciz`: 0個以上の文字列リテラルに終端文字を付けて連続して配置する。

### ラベル
`.L`から始まるラベルはファイルスコープとなり、別ファイルから参照できない。
[5.3 Symbol Names (Using as)](https://sourceware.org/binutils/docs/as/Symbol-Names.html) Local Symbol Names
> A local symbol is any symbol beginning with certain local label prefixes. By default, the local label prefix is ‘.L’ for ELF systems or ‘L’ for traditional a.out systems, but each target may have its own set of local label prefixes. On the HPPA local symbols begin with ‘L$’.

## System V ABI

分割コンパイルをする際はファイル間で引数の受け渡しの形式やデータの配置の仕方などを合わせる必要がある。アセンブリの書き方はOSごとに規約が定められており、これをABI(Application Binary Interface)という。Linux上でx86-64を実行するときはSystem V ABIが使われる。System V ABIには、レジスタの用途、関数の呼び出し規約、オブジェクトファイルおよび実行ファイルのフォーマットなどが定められている。ベースとなるドキュメントとプラットフォーム依存の補遺からなる。

### レジスタ
System V ABIでは汎用レジスタに以下の役割が指定されている。

64bit汎用レジスタ
| 64bitレジスタ | 32bitレジスタ | 16bitレジスタ | 8bitレジスタ | 用途 | 保存 |
|--|--|--|--|--|--|
| RSP | ESP |  SP | SPL | スタックポインタ | callee |
| RBP | EBP |  BP | BPL | ベースポインタ | callee |
| RAX | EAX |  AX |  AL | 返り値 | caller |
| RBX | EBX |  BX |  BL | callee |
| RDI | EDI |  DI | DIL | 第一引数 | caller |
| RSI | ESI |  SI | SIL | 第二引数 | caller |
| RDX | EDX |  DX |  DL | 第三引数 | caller |
| RCX | ECX |  CX |  CL | 第四引数 | caller |
| R8  | R8D | R8W | R8B | 第五引数 | caller |
| R9  | R9D | R9W | R9B | 第六引数 | caller |
| R10 | R10D | R10W | R10B | 一時 | caller |
| R11 | R11D | R11W | R11B | 一時 | caller |
| R12 | R12D | R12W | R12B || callee |
| R13 | R13D | R13W | R13B || callee |
| R14 | R14D | R14W | R14B || callee |
| R15 | R15D | R15W | R15B || callee |

### データアラインメント
多くのCPUではメモリに格納するデータのサイズによって境界が定まっており、アラインメントと呼ばれる。例えば4バイト境界なら4の倍数のアドレスから順にデータを読み出す必要がある。

| 型 | size | alignment |
|--|--|--|
| char | 1 | 1 |
| short | 2 | 2 |
| int | 4 | 4 |
| long | 8 | 8 |
| type* | 8 | 8 |
| type (*)() | 8 | 8 |

アーキテクチャによってアラインメントを揃えることを強制するものとそうでないものがある。x86-64アーキテクチャでは基本的にデータ型のアラインメントを揃える必要はないが、パフォーマンスの観点から揃えることを推奨している。

[AMD64 Architecture Programmer's Manual Volume 1: Application Programming](https://www.amd.com/system/files/TechDocs/24592.pdf) 3.2.5 Data Alignment
> The AMD64 architecture does not impose data-alignment requirements for accessing data in memory. However, depending on the location of the misaligned operand with respect to the width of the data bus and other aspects of the hardware implementation (such as store-to-load forwarding mechanisms), a misaligned memory access can require more bus cycles than an aligned access. For maximum performance, avoid misaligned memory accesses. 

[Intel® 64 and IA-32 Architectures Software Developer’s Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) Volume1: Basic Architecture 4.1.1 Alignment of Words, Doublewords, Quadwords, and Double Quadwords
> Words, doublewords, and quadwords do not need to be aligned in memory on natural boundaries. The natural boundaries for words, double words, and quadwords are even-numbered addresses, addresses evenly divisible by four, and addresses evenly divisible by eight, respectively. However, to improve the performance of programs, data structures (especially stacks) should be aligned on natural boundaries whenever possible.

ABIでは構造体のデータアラインメントが定義されている。
構造体のアラインメントはメンバのアラインメントの最小公倍数となる。データ型は全て2の累乗なので、結果それらの最大値となる。

[System V Application Binary Interface AMD64 Architecture Processor Supplement](https://www.uclibc.org/docs/psABI-x86_64.pdf) 3.1.2 Data Representation
> Structures and unions assume the alignment of their most strictly aligned component. Each member is assigned to the lowest available offset with the appropriate alignment. The size of any object is always a multiple of the object‘s alignment.
> An array uses the same alignment as its elements, except that a local or global array variable of length at least 16 bytes or a C99 variable-length array variable always has alignment of at least 16 bytes.4
> Structure and union objects can require padding to meet size and alignment constraints. The contents of any padding is undefined.

### 関数呼び出し規約
分割コンパイルする際、別のコンパイラでコンパイルしたファイルの関数を用いるためには、関数呼び出し時の引数や返り値を格納するレジスタ/メモリ位置を揃えておく必要がある。最も単純なやり方は引数を全てスタックに格納する方法だが、アクセスの高速なレジスタを用いるために複雑な規約が定められている。

レジスタ
- 関数呼び出しの前後で値を保存しなくてはならない場合、呼び出し側と呼び出し先のどちらで保存するか合わせる必要がある。前者をcaller-saved、後者をcallee-savedという。
- caller-saved: rax, rdi, rsi, rdx, rcx, r8, r9, r10, r11
- callee-saved: rsp, rbp, rbx, r12, r13, r14, r15
-  DFは関数の開始と終了時に順方向にリセットする。

スタックフレーム(引数が整数の場合)
| アドレス | 値 |
|--|--|
| rsp + 32 | 引数8 |
| rsp + 16 | 引数7 |
| rsp + 8  | リターンアドレス |
| rsp      | rbpの値 |

スタック
- スタックアラインメント: callの直前、rspは16の倍数になっている必要がある。

引数渡し
- 整数はrdi, rsi, rdx, rcx, r8, r9に左から順に格納し、それ以外はスタックに積む。

### ELFファイル
ELFは少なくとも三つのセグメントに分かれており、text, data, bssの順に並んでいる。bssセグメントには0に初期化されるデータが格納されている。

## x86-64アーキテクチャ

### レジスタ
64bit汎用レジスタ
| 64bitレジスタ | 32bitレジスタ | 16bitレジスタ | 8bitレジスタ |
|--|--|--|--|
| RSP | ESP |  SP | SPL |
| RBP | EBP |  BP | BPL |
| RAX | EAX |  AX |  AL |
| RBX | EBX |  BX |  BL |
| RDI | EDI |  DI | DIL |
| RSI | ESI |  SI | SIL |
| RDX | EDX |  DX |  DL |
| RCX | ECX |  CX |  CL |
| R8  | R8D | R8W | R8B |
| R9  | R9D | R9W | R9B |
| R10 | R10D | R10W | R10B |
| R11 | R11D | R11W | R11B |
| R12 | R12D | R12W | R12B |
| R13 | R13D | R13W | R13B |
| R14 | R14D | R14W | R14B |
| R15 | R15D | R15W | R15B |

フラグレジスタ
| 64bitレジスタ | 32bitレジスタ | 16bitレジスタ |
| RFLAGS | EFLAGS | FLAGS |

| bit | ニーモニック | 説明 | 条件 |
|--|--|--|--|
| 11 | OF |        Overflow Flag | 符号付き整数演算でオーバーフロー |
| 10 | DF |       Direction Flag | 文字列操作のデータポインタが減る向き |
|  7 | SF |            Sign Flag | 算術演算の結果が負 |
|  6 | ZF |            Zero Flag | 算術演算の結果が0 |
|  4 | AF | Auxiliary Carry Flag |  |
|  2 | PF |          Parity Flag | 演算結果の最下位byteの1が偶数個 |
|  0 | CF |           Carry Flag | 算術演算で最上位bitで繰り上がり/繰り下がりが発生 |

DFはcontrol flag、他の6個はstatus flagと呼ばれている。

128bitSSE(Streaming SIMD Extended)レジスタ
80bit x87浮動小数点レジスタ

SSEレジスタはvector registerとも呼ばれる。

### メモリモデル
メモリは1byteごとに並んでいる。メモリには大きく四つの領域がある。プログラムそのものとプログラムが扱うデータはどちらもメモリに入っている。

AMD64はリトルエンディアン(低ビットが低アドレス)なので、例えば32bit以下の64bit整数を`DWORD PTR [rax]`で読んでも`QWORD PTR [rax]`で読んでも全く問題ない。

64bitモードでは、デフォルトのオペランドサイズは基本的に4byteであり、REXプリフィックスの付与された命令では8byteである。ただし、暗黙にスタックポインタを参照する命令についてはデフォルトで8byteオペランドサイズを採用する。

[AMD64 Architecture Programmer's Manual Volume 1: Application Programming](https://www.amd.com/system/files/TechDocs/24592.pdf) 3.2.3.1 Default Operand Size
> There are several exceptions to the 32-bit operand-size default in 64-bit mode, including near branches and instructions that implicitly reference the RSP stack pointer. For example, the near CALL, near JMP, Jcc, LOOPcc, POP, and PUSH instructions all default to a 64-bit operand size in 64-bit mode. Such instructions do not need a REX prefix for the 64-bit operand size. For details, see “GeneralPurpose Instructions in 64-Bit Mode” in Volume 3. 

### アドレッシング
メモリにアクセスする方法をアドレシングという。実効アドレスの生成には以下の5つの方法がある。

- 完全なアドレス: Base + Displacement(Offset)
- RIP相対アドレシング: IP(PC) + Displacement
- インデックス付きレジスタ間接: base + index * scale + disp, scale = 1, 2, 4, 8
- スタックアドレス
- string address

### 命令セット
64bitモードではデフォルトのアドレスサイズは64bitである。

#### データ転送
レジスタ/メモリからレジスタ/メモリへデータを移動する。ただしメモリからメモリへの移動はできない。第二オペランドとして即時定数(immediate constant)を用いることができる。
- MOV: 第二オペランドから第一オペランドにデータを移動する。オペランドは同じ大きさ。
- MOVZX: ゼロ拡張(zero extention)。ソースより大きいレジスタに転送する際、上位bitをゼロ埋めする。
- MOVSX: 符号拡張(sign extention)。ソースより大きいレジスタに転送する際、符号を考慮して上位bitに拡張する。
- CQO: RAXを符号拡張してRDX:RAXに格納。

#### スタック操作
- push: スタックポインタをデータサイズ分減らし、スタックトップにデータをコピーする。
- pop: スタックトップからデータをコピーし、スタックポインタをデータサイズ分増やす。

#### 有効アドレス
- lea: load effective addressの略。

#### 算術演算
- ADD a b: a += b
- SUB a b: a -= b
- IMUL a b: a *= bの符号あり乗算。
- IDIV a: RDX:RAX / aの符号あり除算。商と剰余をそれぞれRAX, RDXに格納する。
- NEG: 2の補数を求める。
- INC: インクリメント。CFは影響しない。
- DEC: デクリメント。CFは影響しない。

#### 比較演算
- CMP a b: a - bを計算しフラグレジスタを更新。

#### 条件
条件にしたがって8bitオペランドに0, 1を設定。
- SETE r/m8: ZF == 1
- SETNE r/m8: ZF == 0
- SETL r/m8: SF != OF
- SETLE r/m8: SF != OF or ZF == 1
- SETG r/m8: SF == OF and ZF == 0
- SETGE r/m8: SF == OF

#### 制御転送
条件ジャンプ
- JE rel8: ZF == 1
- JNE rel8: ZF == 0
- JL rel8: SF != OF
- JLE rel8: SF != OF or ZF == 1
- JG rel8: SF == OF and ZF == 0
- JGE rel8: SF == OF

無条件ジャンプ
- JMP

関数呼び出し
- CALL: プログラムカウンタをスタックにpushしてジャンプ。
- RET: スタックをpopしてそのアドレスにジャンプ。

制御転送命令はスタックアラインメントが適切でない場合、著しくパフォーマンスが落ちる。

[AMD64 Architecture Programmer's Manual Volume 1: Application Programming](https://www.amd.com/system/files/TechDocs/24592.pdf) 3.7.3.1 Stack Alignment
> Control-transfer performance can degrade significantly when the stack pointer is not aligned properly. Stack pointers should be word aligned in 16-bit segments, doubleword aligned in 32-bit segments, and quadword aligned in 64-bit mode. 

[Intel® 64 and IA-32 Architectures Software Developer’s Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) Volume1: Basic Architecture 6.2.2 Stack Alignment
> The stack pointer for a stack segment should be aligned on 16-bit (word) or 32-bit (double-word) boundaries, depending on the width of the stack segment. The D flag in the segment descriptor for the current code segment sets the stack-segment width (see “Segment Descriptors” in Chapter 3, “Protected-Mode Memory Management,” of the Intel® 64 and IA-32 Architectures Software Developer’s Manual, Volume 3A). The PUSH and POP instructions use the D flag to determine how much to decrement or increment the stack pointer on a push or pop operation, respectively. When the stack width is 16 bits, the stack pointer is incremented or decremented in 16-bit increments; when the width is 32 bits, the stack pointer is incremented or decremented in 32-bit increments. Pushing a 16-bit value onto a 32-bit wide stack can result in stack misaligned (that is, the stack pointer is not aligned on a double word boundary). One exception to this rule is when the contents of a segment register (a 16-bit segment selector) are pushed onto a 32-bit wide stack. Here, the processor automatically aligns the stack pointer to the next 32-bit boundary

> The processor does not check stack pointer alignment. It is the responsibility of the programs, tasks, and system procedures running on the processor to maintain proper alignment of stack pointers. Misaligning a stack pointer can cause serious performance degradation and in some instances program failures.

#### システムコール
- syscall
- sysret
エラーメッセージ

### 浮動小数点命令

#### ロード
スタックトップにロード
- fld: Floating-Point Load
- fldz: +0.0
- fld1: +1.0
- fldpi: 円周率$\pi$
- fldl2e: $\log_2 e$
- fldl2t: $\log_2 10$
- fldlg2: $\log_{10}2$
- fldln2: $\log_e 2$

#### 算術演算
四則演算
- fadd: 加算
- fsub: 減算
- fmul: 乗算
- fdiv: 除算

- frndint: 整数に丸める

平方根
- fsqrt: 平方根

超越関数
- fsin: 正弦
- fcos: 余弦
- fsincos: スタックトップの値をsinで置換し、cosをプッシュする。sinとcosは同時に計算される。
- fptan: スタックトップをtanで置換し、1.0をプッシュする。
- fpatan: arctan(ST(1)/ST(0))をST(1)にコピーし、スタックをポップする。

対数関数
- f2xm1: $ST(0) = 2^{ST(0)} - 1 (-1 \leq ST(0) \leq 1)$
- fscale: $ST(0) = ST(0) \times 2^{\left[ST(1)\right]}$
- fyl2x: $ST(1) = ST(1) \times \log_2{ST(0)} (ST(0) > 0)$とした後、スタックをポップする。
- fyl2xp1: $ST(1) = ST(1) \times \log_2{ST(0) + 1} (0 < |ST(0)| < 1 - \sqrt{2}/2)$とした後、スタックをポップする。

## コンパイラの構成

- 字句解析: 文字列をトークン列に変換。
- プリプロセス: マクロやヘッダファイルの読み込みなどの前処理を行う。
- 構文解析: トークン列を抽象構文木に変換。
- 意味解析: 静的意味の解析。変数のスコープや型検査、制御の流れの検査を行う。
- 中間コード生成: やってない。
- 最適化: やってない。
- コード生成: アセンブリを生成。

```c main.c
int main(int argc, char **argv){
    if(argc != 2){
        fprintf(stderr, "incorrect number of arguments");
        return 1;
    }

    file_name = argv[1];
    code_head = read_file(file_name); // ファイルの先頭のポインタを返す
    token *token_head = tokenize(code_head); // トークン列の先頭を返す
    node *node_head = program(token_head); // 構文木のルートの列を返す
    gen_code(node_head); // コード生成
}
```

<!-- ### コンパイル時に決定できること
- コンパイル時に決定できる
    - 変数の個数
        - 関数内で定義される変数は構文解析の段階でカウントしておいて、rbp直下にまとめて確保する。
    - 変数のアドレス
        - 変数名はアセンブリの段階で消える。
    - 変数が定義済みかどうか
- コンパイル時に決定できない
    - 変数の値
    - rspの値
        - アラインメントはアセンブリで調整する。 -->

## 字句解析

ソースコードをトークン列に分解する。トークンとはプログラムにおいて意味のある最小単位のこと。トークンは通常正規表現として定義され、決定性有限オートマトンで受理できる。一般に正規表現にマッチする部分文字列は複数あるが、通常は最長一致の原則が適用される。例えば`--`は`--`と判定され、`- -`は`-`, `-`と分割される。

C言語のソースコードは

- キーワード(keyword): 型名、制御構造など
- 識別子(identifier): 変数名、関数名など
- 定数(constant)
- 文字列リテラル(string-literal)
- 区切り文字(punctuator)

に加えて、空白('\t', '\n', '\v', '\f', '\r', ' ')とコメント文から成る。

### トークナイザ

トークンを前方一致した時点で決定できるものとそうでないものに分ける。括弧や演算子は一致した時点で確定するので前者、keywordsは文字列が一致してかつ、直後にアルファベット、数字、'_'以外の文字が来る必要があるので後者。

```c tokenize.c
char *types[17] = {"extern", "const", "volatile", "static", "signed", "unsigned", "void", "_Bool", "char", "short", "int", "long", "float", "double", "struct", "union", "enum"};
char *keywords[12] = {"include", "typedef", "return", "if", "else", "switch", "case", "while", "for", "continue", "break", "sizeof"};
char *puncts[41] = {
    "+=", "-=", "*=", "/=", "%=", "||", "&&", "==", "!=", "<=", ">=", "<<", ">>", "++", "--", "->", 
    "#", "=", "?", ":", "|", "^", "&", "<", ">", "+", "-", "*", "/", "%", "&", "!", ".", ",", ";", "(", ")", "{", "}", "[", "]"
};
```

トークン列を連結リストとして定義。トークンの文字列の情報はまだ必要なので持っておく。

```dcl.h
struct token {
    token_kind kind;
    token *next;
    char *str;
    int len;
    int val;
};
```

punctsとkeywordsの違いは字句解析後必要ないためTK_RESERVEDにまとめる。

```c dcl.h
typedef enum {
    TK_RESERVED,
    TK_TYPE,
    TK_ID,
    TK_NUM,
    TK_CHAR,
    TK_STRING,
    TK_EOF,
} token_kind;
```

## 構文解析

トークン列から抽象構文木(Abstract Syntax Tree)を構築する。構文は通常文脈自由文法として定義される。一般の文脈自由文法の構文解析には$O(N^3)$かかるので、普通のプログラミング言語は構文解析が$O(N)$で済むLL(1)文法かそれに近い文法で定義されている。

文脈自由文法は普通BNF(Bakcus-Naur Form)やその拡張であるEBNF(Extended BNF)で定義される。

### 構文
実装するC言語のサブセットのEBNF。

```
program = ext*
ext     = type_head (type_ident "=" initializer) ("," type_ident "=" initializer)* ";"
        | type_head type_ident "{" stmt* "}"
        | "typedef" type_ident ";"
dcl     = type_head (type_ident "=" initializer) ("," type_ident "=" initializer)* ";"
type_head = ("const" | "volatile" | "static")* (
          "void"
        | "_Bool"
        | ("unsigned" | "signed") ("char" | "short" ("int")? | "int" | "long" ("long")? ("int")?)
        | ("char" | "short" ("int")? | "int" | "long" ("long")? ("int")? | "long" "double")
        | "struct" ident? "{" (type_head type_ident ("," type_ident)* ";")* "}"
        | "enum" ident? "{" ident ("=" cond)? ("," ident ("=" cond)?)* (",")?  "}"
type_ident = type_head ("*")* ("(" type_ident ")" | ident)? (("[" assign? "]")* | "(" (type_ident ("," type_ident)* ("...")?)? ")")
initializer = "{" initializer ("," initializer)* (",")? "}"
            |  assign
stmt    = ";"
        | expr ";"
        | "if" "(" expr ")" stmt ("else" stmt)?
        | "switch" "(" expr ")" stmt
        | "case" primary ":" stmt
        | "while" "(" expr ")" stmt
        | "for" "(" (dcl | expr? ";") expr? ";" expr? ")" stmt
        | "continue" ";"
        | "break" ";"
        | "return" expr? ";"
        | "{" (dcl | stmt)* "}"
expr    = assign ("," assign)?
assign  = cond ("=" assign | "+=" assign | "-=" assign | "*=" assign | "/=" assign | "%=" assign)?
cond    = log_or ("?" expr ":" cond)?
log_or  = log_and ("||" log_and)*
log_and = bit_or ("&&" bit_or)*
bit_or  = bit_xor ("|" bit_xor)*
bit_xor = bit_and ("^" bit_and)*
bit_and = equal ("&" bit_and)*
equal   = relat ("==" relat | "!=" relat)*
relat   = shift ("<" shift | "<=" shift | ">" shift | ">=" shift)*
shift   = add ("<<" add | ">>" add)*
add     = mul ("+" mul | "-" mul)*
mul     = unary ("*" unary | "/" unary ? "%" unary)*
unary   = primary
        |"+" unary
        | "-" unary
        | "&" unary
        | "*" unary
        | "++" unary
        | "--" unary
        | "sizeof" unary
        | "(" type_ident ")" unary
primary = ("(" expr ")" | ident | num | char | string)
        ("[" assign "]" | "(" (assign ("," assign)*)? | "." ident | "->" ident | "++" | "--")*
```

パーサは再帰下降構文解析で行う。再帰下降構文解析では非終端記号と解析関数が一致するので、人間が書くのに適している。ここでは構文解析と意味解析を同時に行う。

### ノード
構文木のノードを分類し、子要素を持つ。変数名や関数名以外のトークンの文字列は消去する。型のノードは作らず、修飾子ノードの属性として付与する。

```c dcl.h
typedef enum {
    // definition
    ND_FUNC_DEF,
    ND_GLOBAL_DEF,
    ND_LOCAL_CONST,

    // statement
    ND_BLOCK,
    ND_IF,
    ND_SWITCH,
    ND_CASE,
    ND_DEFAULT,
    ND_WHILE,
    ND_FOR,
    ND_CONTINUE,
    ND_BREAK,
    ND_RET,

    // binary operator
    ND_COMMA,
    ND_ASSIGN,
    ND_COND,
    ND_LOG_OR,
    ND_LOG_AND,
    ND_BIT_OR,
    ND_BIT_XOR,
    ND_BIT_AND,
    ND_EQ,
    ND_NEQ,
    ND_LT,
    ND_LEQ,
    ND_LSHIFT,
    ND_RSHIFT,
    ND_ADD,
    ND_SUB,
    ND_MUL,
    ND_DIV,
    ND_MOD,

    // unary operator
    ND_NEG,
    ND_LOG_NOT,
    ND_ADR,
    ND_DEREF,
    ND_CAST,

    // primary
    ND_GLOBAL,
    ND_LOCAL,
    ND_NUM,
    ND_STRING,
    ND_FUNC_CALL,
    ND_DOT,
    ND_ARROW,
} node_kind;
```

```c dcl.h
struct node {
    node_kind kind;
    node *op1, *op2, *op3, *op4;
    node *head;
    node *next;
    type *ty;
    char *name;
    int len;
    int offset;
    int val;
};
```

### シンボルテーブル
ソースコードを前から読み込む際、現在定義されている識別子を管理するシンボルテーブルを作る。シンボルテーブルをスタックで実装するとき、識別子が登場する度に未宣言であることを確認してpushし、ブロックを抜ける際にpopすると、静的スコープが実現できる。

シンボルテーブルは、ローカル変数、グローバル変数、タグ名の3つを作る。ローカル変数で名前解決できなかった場合にグローバル変数で名前解決を試みる。識別子と加える表は以下のようになる。

- ローカル変数表: ローカル変数名
- グローバル変数表: グローバル変数名、関数名、typedefで定義された型名、列挙型のメンバ
- タグ表: 構造体と列挙型のタグ名

### 型
型の種類を以下のように分類する。

```c dcl.h
typedef enum {
    NOHEAD,
    VOID,
    BOOL,
    CHAR,
    SHORT,
    INT,
    LONG,
    PTR,
    ARRAY,
    FUNC,
    STRUCT,
    ENUM,
} type_kind;
```

型を表す構造体を用意する。ポインタ、配列の要素の型、関数の返り値は`ptr_to`に格納する。

```c dcl.h
struct type {
    type_kind kind;
    type *ptr_to;
    symb *head;
    int size;
    int align;
    char *name;
    int len;
};
```

Cの型はネストする部分が不完全になっているので、不完全な部分を仮にNOHEADとして読む。

例えば`int *(*a)[]`のような型を考える。外側とネストしている部分をそれぞれ

- `int *?[]`: ARRAY -> PTR -> INT
- `(*a)`: PTR -> NOHEAD

と読む。全体の型はNOHEADの部分に最初に読んだ部分を代入して、PTR -> ARRAY -> PTR -> INTとなる。全体の型が決定した後、以下のような無効な型を排除する。

- VOID
- ? -> ARRAY -> VOID -> ?
- ? -> FUNC -> ARRAY -> ?

### 構造体と列挙型
構造体や列挙型にタグがある場合はタグを格納するシンボルテーブルに登録される。つまりグローバル変数やローカル変数と重複して構わない。
相互参照するような型を作る場合、一方の型が未定義とならないようにタグ名だけの宣言が可能。つまり`struct a;`や`enum a;`と書くとき`a`は未宣言でも良い。構造体は自分自身を含むことはないが、自分自身へのポインタは含むことができる。

構造体のメンバが定義される場合、それぞれのメンバの名前と型を持っておく。列挙型のメンバが定義される場合、メンバ名とその値を持っておく。列挙型のメンバはコンパイル時に整数に置き換えられる。トップレベルで宣言された列挙型のメンバ名はグローバル変数表に加える。

列挙型のメンバは`cond`で初期化できるが、コンパイル時に決定できる必要があるのでその場で評価する。

```c parse.c
while(!expect("}")){
    token *tk = cur;
    cur = cur->next;
    if(expect("=")) i = eval_const(cond());
    push_enum(tk->str, tk->len, i);
    i++;
    if(expect(",")) continue;
    if(expect("}")) break;
    error(cur, "expected ',' or '}'");
}
```

### プログラム
トップレベルには関数定義とグローバル変数の宣言が並ぶ。パーサは外部宣言のキューを作りその先頭要素を返す。

```c parse.c
node *program(token *token_head){
    cur = token_head;
    node_head = calloc(1, sizeof(node));
    node_tail = node_head;

    while(cur->kind != TK_EOF){
        ext();
    }
    return node_head->next;
}
```

### 関数定義
関数名はラベルとしてアセンブリ上に残るが、関数の引数はローカル変数として扱える。したがって、関数定義ノードは関数名、型、処理内容、関数全体で定義されるローカル変数全体のオフセットを持っておく必要がある。

1. 関数名をグローバル変数表に登録。
2. ローカル変数表を初期化。ローカル変数の個数と全体のオフセットを0で初期化。
3. 引数をローカル変数として登録。
4. 処理内容をパース。

### グローバル変数宣言
グローバル変数ノードは変数名と型、そして初期値を持っておく。

グローバル変数は宣言が重複しても構わない(標準ライブラリで重複した宣言が存在)。
```c
void push_global(symb_kind kind, symb *sy){
    for(symb *var = global_head; var; var = var->next){
        if(sy->len == var->len && memcmp(sy->name, var->name, var->len) == 0){
            return;
        }
    }

    // 省略
}
```

### ローカル変数宣言
ローカル変数がスコープ内で使用されるためには、ブロック文の中で宣言される必要がある。C言語の場合、ブロック文の中にないローカル変数宣言は構文エラーとなる。

1. ローカル変数表に同じ変数名がないことを確認。
2. 型のサイズとアラインメントからオフセットを計算。
3. 現在の関数のローカル変数の個数とオフセットを更新。

```c
void push_local(symb *sy){
    for(symb *var = local_head; var; var = var->next){
        if(sy->len == var->len && memcmp(sy->name, var->name, var->len) == 0){
            error(cur, "this variable is already declared");
        }
    }
    symb *var = calloc(1, sizeof(symb));
    var->next = local_head;
    var->ty = sy->ty;
    var->name = sy->name;
    var->len = sy->len;
    var->offset = local_head->offset + size_of(sy->ty);
    var->offset = (var->offset + align_of(sy->ty) - 1) / align_of(sy->ty) * align_of(sy->ty);

    local_head = var;
    local_num++;
    if(max_offset < var->offset) max_offset = var->offset;
}
```

### 文(Statement)
文とは手続きであり、式に`;`を付けたものや`return`文、制御文、ブロック構文がある。

[N1570](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf) 6.8 Statements and blocks
```
statement:
    labeled-statement
    compound-statement
    expression-statement
    selection-statement
    iteration-statement
    jump-statement
```

```
stmt    = ";"
        | expr ";"
        | "if" "(" expr ")" stmt ("else" stmt)?
        | "switch" "(" expr ")" stmt
        | "case" primary ":" stmt
        | "while" "(" expr ")" stmt
        | "for" "(" (dcl | expr? ";") expr? ";" expr? ")" stmt
        | "continue" ";"
        | "break" ";"
        | "return" expr? ";"
        | "{" (dcl | stmt)* "}"
```

#### ブロック文
1. 現在のローカル変数のカウント
2. ブロック内の処理をパース
3. ローカル変数表を元に戻す。

```c
else if(expect("{")){
    nd->kind = ND_BLOCK;
    nd->head = calloc(1, sizeof(node));
    node *item = nd->head;
    int num = local_num;
    while(!expect("}")){
        if(find_type()) item->next = dcl();
        else item->next = stmt();
        item = item->next;
    }
    nd->head = nd->head->next;
    pop_local(num);
}
```

#### 制御文
`"else if"`というキーワードはなく、`"else" stmt`は`"else" "{" stmt "}"`と同じ意味となる。

ぶら下がりelse(dangling else)とは、`else`と対応する`if`が判定できず、文が曖昧になる問題。解決策として、最も近くにある対応していない`if`と結合することにすれば良い。

```c
if(a) if(b) s1; else s2;
```

は

```c
if(a){
    if(b) s1;
    else s2;
}
```

と解釈される。

`if`文を読んだ後に`else`が来る場合、`if else`として解釈すれば自然に実装できる。

```c parse.c
else if(expect("if")){
    if(!expect("(")) error(cur, "expected '('");

    nd->kind = ND_IF;
    nd->op1 = expr();

    if(!expect(")")) error(cur, "expected ')'");

    nd->op2 = stmt();

    if(expect("else")) nd->op3 = stmt();
}
```

- `switch-case`文: 現在そのブロック内にいる`switch`文のリストをスタックで管理する。`switch`文は対応する`case`の式を持っておき、`case`文はルート方向に最も近い`switch`文とその`switch`文で何番目の`case`文であるかを持っておく。
- `for`文: 抜けるとき、括弧内で宣言された変数をシンボルテーブルから削除する。

### 式(Expression)

```
expr    = assign ("," assign)?
assign  = cond ("=" assign | "+=" assign | "-=" assign | "*=" assign | "/=" assign | "%=" assign)?
cond    = log_or ("?" expr ":" cond)?
log_or  = log_and ("||" log_and)*
log_and = bit_or ("&&" bit_or)*
bit_or  = bit_xor ("|" bit_xor)*
bit_xor = bit_and ("^" bit_and)*
bit_and = equal ("&" bit_and)*
equal   = relat ("==" relat | "!=" relat)*
relat   = shift ("<" shift | "<=" shift | ">" shift | ">=" shift)*
shift   = add ("<<" add | ">>" add)*
add     = mul ("+" mul | "-" mul)*
mul     = unary ("*" unary | "/" unary ? "%" unary)*
unary   = primary
        |"+" unary
        | "-" unary
        | "&" unary
        | "*" unary
        | "++" unary
        | "--" unary
        | "sizeof" unary
        | "(" type_ident ")" unary
primary = ("(" expr ")" | ident | num | char | string)
        ("[" assign "]" | "(" (assign ("," assign)*)? | "." ident | "->" ident | "++" | "--")*
```

#### 演算
演算は基本的に演算子ごとにノードを用意する。

加算は次のように再帰的に定義されている。

[N1570](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf) 6.5.6 Additive operators
```
additive-expression:
    multiplicative-expression
    additive-expression + multiplicative-expression
    additive-expression - multiplicative-expression
```

再帰的な定義は演算子の結合性を表現するのに都合が良い。しかし左結合の演算を左再帰的に定義する場合、プログラムで左から構文解析すると無限ループとなる。そこで次のような左再帰を除去した等価な文法に書き換える。

```
add = mul ("+" mul | "-" mul)*
```

これを実装すると次のようになる。

```c: parse.c
node *add(){
    node *nd = mul();
    while(true){
        if(expect("+")){
            nd = node_binary(ND_ADD, nd, mul());
            continue;
        }
        if(expect("-")){
            nd = node_binary(ND_SUB, nd, mul());
            continue;
        }
        return nd;
    }
}
```

#### `sizeof`演算子
`sizeof`演算はコンパイル時に計算され、定数ノードとして解析される。

```c: parse.c
node *unary(){
    // 略
    if(expect("sizeof")){
        if(expect("(")){
            type *ty;
            if(find_type()) ty = type_ident()->ty;
            else ty = unary()->ty;
            if(!expect(")")) error(cur, "expected ')'");
            return node_num(type_base(INT), size_of(ty));
        }else{
            return node_num(type_base(INT), size_of(unary()->ty));
        }
    }
    // 略
}
```

#### 識別子
シンボルテーブルにアクセスして名前解決を試みる。ローカル変数なら型とオフセット、グローバル変数なら型と名前をコピーしてノードを作る。






## コード生成

構文解析によって得たASTを深さ優先探索する。

<!-- 式(expression)や`return`はオペランドをスタックからpopし、返り値をpushするようにしておく(連続する操作でpop, push命令が連続することになるが、そういった最適化は後回しにする)。それ以外の結果を返さないノードではスタックを空にして収支を合わせるようにする。 -->
基本的にスタック操作は8byte単位で行い、レジスタとメモリ間でデータを移動するときのみデータサイズを考慮する。

### 型
データ型によってメモリから読むサイズ及びレジスタのサイズを変える。

```c codegen.c
char *rax_[4] = { "al",  "ax", "eax", "rax"};
char *rbx_[4] = { "bl",  "bx", "ebx", "rbx"};

char *rdi_[4] = {"dil",  "di", "edi", "rdi"};
char *rsi_[4] = {"sil",  "si", "esi", "rsi"};
char *rdx_[4] = { "dl",  "dx", "edx", "rdx"};
char *rcx_[4] = { "cl",  "cx", "ecx", "rcx"};
char  *r8_[4] = {"r8b", "r8w", "r8d", "r8" };
char  *r9_[4] = {"r9b", "r9w", "r9d", "r9" };

char **int_arg_reg[6] = {rdi_, rsi_, rdx_, rcx_, r8_, r9_};
```

```c codegen.c
void mov_memory_from_register(char *dest[4], char *src[4], type *ty){
    switch(ty->kind){
        case BOOL:
        case CHAR:
            printf("    mov BYTE PTR [%s], %s\n", dest[3], src[0]);
            break;
        case SHORT:
            printf("    mov WORD PTR [%s], %s\n", dest[3], src[1]);
            break;
        case INT:
            printf("    mov DWORD PTR [%s], %s\n", dest[3], src[2]);
            break;
        case LONG:
        case PTR:
            printf("    mov QWORD PTR [%s], %s\n", dest[3], src[3]);
            break;
    }
}
```

```c codegen.c
mov_memory_from_register(rax_, int_arg_reg[num_param_int], param->ty);
```

### 関数定義
`rbp`レジスタはcallee-savedであり、ベースポインタはcalleeが保存する。

1. 関数プロローグ: ベースポインタを更新。関数内の局所変数の分のメモリを確保。
2. 次の関数呼び出しに備えて、引数をレジスタまたはスタックからrbp直下のスタック領域に移動。
3. 処理内容をコード生成。
4. 関数エピローグ: ベースポインタを戻す。返り値を設定。

第一引数、第二引数を`[rbp-8]`, `[rbp-16]`,...に格納する。

| アドレス | 値 | 命令 |
|--|--|--|
| rbp + 32 | 引数8 ||
| rbp + 16 | 引数7 ||
| rbp +  8 | リターンアドレス | call func |
| rbp      | callerのrbp | push rbp |
| rbp -  8 | ローカル変数1 ||
| rbp - 16 | ローカル変数2 ||

```c codegen.c
// 略

int num_param_int = 0;
for(symb *param = nd->ty->head; param; param = param->next){
    if(num_param_int < 6){
        printf("    lea rax, [rbp-%d]\n", param->offset);
        mov_memory_from_register(rax_, int_arg_reg[num_param_int], param->ty);
    }else{
        printf("    lea rax, [rbp-%d]\n", param->offset);
        printf("    mov rbx, [rbp+%d]\n", 8 * (num_param_int - 4));
        mov_memory_from_register(rax_, rbx_, param->ty);
    }
    num_param_int++;
}

gen_stmt(fn->stmt);

// 略
```

### 文(Statement)
#### 制御文
制御文はラベルを貼ってジャンプ命令で遷移する。

```c codegen.c
case ND_WHILE:
    l1 = label_num;
    l2 = label_num + 1;
    label_num += 2;
    push_block(ND_WHILE, l1, l2);

    printf(".L%d:\n", l1);

    gen_expr(nd->op1);
    printf("    pop rax\n");
    printf("    cmp rax, 0\n");
    printf("    je .L%d\n", l2);
    gen_stmt(nd->op2);
    printf("    jmp .L%d\n", l1);
    
    printf(".L%d:\n", l2);

    pop_block();
    return;
```

#### `return`文
`ret`の直前に返り値を設定し、ベースポインタを元に戻す。返り値がない場合は0を設定する。

### 式(Expression)
#### スタックマシン
式はスタックマシンで計算する。つまり演算を実行する直前で、オペランドがスタックに積まれているようにし、スタックからレジスタに値をpop、演算の後再び結果をスタックにpushする。例えば加算は以下のようになる。

#### 左辺値と右辺値
- 配列要素は順番に格納され、大きさの情報は実行時に失われる。
- 配列はその先頭要素のポインタに暗黙にキャストされるが、ポインタの実体があるわけではない。
- 配列のアドレスは先頭要素のアドレスと等しい。

- 構造体メンバは順番に格納され、それぞれのアドレスを保持する。
- 構造体のアドレスは先頭要素のアドレスと等しい。

`lea`命令は一命令でbase + scale * index + dispを計算するので、算術命令でアドレスを計算するより速い。

#### 代入
左辺は左辺値として展開し、右辺は右辺値として展開する。

```c codegen.c
case ND_ASSIGN:
    gen_lval(nd->op1);
    gen_expr(nd->op2);

    printf("    pop rdi\n");
    printf("    pop rax\n");
    mov_memory_from_register(rax_, rdi_, nd->ty);
    printf("    push rdi\n");
    break;
```

#### 論理演算
論理演算は短絡評価されるので、左オペランドを計算した時点で結果が定まる場合、右オペランドは計算せずにジャンプする。C言語では0以外はtrueの扱いなので比較の度に`setne`で0, 1を格納し直す。

```c codegen.c
case ND_LOG_OR:{
    int label_end = label_num++;

    gen_expr(nd->op1);
    printf("    pop rax\n");
    printf("    cmp rax, 0\n");
    printf("    setne al\n");
    printf("    movzb rax, al\n");
    printf("    jne .L%d\n", label_end);

    gen_expr(nd->op2);
    printf("    pop rax\n");
    printf("    cmp rax, 0\n");
    printf("    setne al\n");
    printf("    movzb rax, al\n");

    printf(".L%d:\n", label_end);
    printf("    push rax\n");
    return;
}
```

#### 算術演算
`idiv`命令は`rdx:rax`を被除数として計算するので、その前に`cqo`命令で`rax`を符号拡張する。

```
    cqo
    idiv rdi
    push rax
```

#### 関数呼び出し
1. スタックアラインメントを調整。
2. 引数を後ろから計算。
3. 6個までの引数はレジスタにpop。
4. `al`を使用するベクトルレジスタの数以上に設定。
5. `call`命令。
6. スタックアラインメントを戻す。

## セルフホスト
### 方針
コンパイル以外のプリプロセス、アセンブル、リンクはgccに任せる。

### 前処理
gccのプリプロセッサはマクロを置換する際に参照したファイルの名前と行を記載する。これはlinemarkerと呼ばれる。

[The C Preprocessor 9 Preprocessor Output](https://gcc.gnu.org/onlinedocs/cpp/Preprocessor-Output.html)

よってコンパイル時にlinemarkerを飛ばす。
```c tokenize.c
// skip linemarker
if(*p == '#'){
    while(*p != '\n'){
        p++;
    }
    continue;
}
```

更にgcc固有の機能を飛ばす。
```c tokenize.c
if(memcmp(p, "__attribute__", 13) == 0){
    while(*p != ';') p++;
    continue;
}
if(memcmp(p, "__extension__", 13) == 0){
    p += 13;
    continue;
}
if(memcmp(p, "__inline", 8) == 0){
    p += 8;
    continue;
}
if(memcmp(p, "__asm__", 7) == 0){
    while(*p != '\n') p++;
    continue;
}
if(memcmp(p, "__restrict", 10) == 0){
    p += 10;
    continue;
}
```

gccはファイル内では宣言されないビルトインの型や変数があるので、予め変数表に登録しておく。
```c parse.c
void add_builtin(symb_kind kind, char *name){
    symb *sy = calloc(1, sizeof(symb));
    sy->kind = kind;
    sy->next = NULL;
    sy->ty = type_base(NOHEAD);
    sy->name = name;
    sy->len = strlen(name);
    push_global(kind, sy);
}
```

```c parse.c
// implicit declaration of builtin type or function
add_builtin(SY_TYPE, "__builtin_va_list");
add_builtin(SY_VAR, "__builtin_va_start");
add_builtin(SY_VAR, "__builtin_bswap16");
add_builtin(SY_VAR, "__builtin_bswap32");
add_builtin(SY_VAR, "__builtin_bswap64");
```

更にインライン展開されてヘッダファイル内で定義されている関数が存在する。
```c parse.c
if(var->len == 10 && memcmp(var->name, "__bswap_16", 10) == 0){ stmt(); return; }
if(var->len == 10 && memcmp(var->name, "__bswap_32", 10) == 0){ stmt(); return; }
if(var->len == 10 && memcmp(var->name, "__bswap_64", 10) == 0){ stmt(); return; }
if(var->len == 17 && memcmp(var->name, "__uint16_identity", 17) == 0){ stmt(); return; }
if(var->len == 17 && memcmp(var->name, "__uint32_identity", 17) == 0){ stmt(); return; }
if(var->len == 17 && memcmp(var->name, "__uint64_identity", 17) == 0){ stmt(); return; }
```

### セルフホストの確認
gccでコンパイルした自作コンパイラをstage1という。そして、stage1で自分自身をコンパイルしたものをstage2、stage2で自分自身をコンパイルしたものをstage3と呼ぶ。stage2とstage3はコンパイラとソースコードが同じなので出力は完全に等しくなる。

しかしgccでリンクした場合、アセンブリが同一でも実行ファイルが同じとは限らない。

[Linuxのコンパイル結果が同等であることの確認方法](http://seigaji.info/wordpress/2015/02/15/linux_elf_binary_diff/)
> まず差分が発生する理由として考えられるのがコンパイラがビルドする際にリンカが使うためのシンボル情報を付加しているというのは常識。ということでstripコマンドを使ってみることにしました。

リンカによって付加されるシンボル情報が異なるらしい。つまり、
```
$ strip -s stage2
$ strip -s stage3
$ diff stage2 stage3
```
で何も出なければセルフホスト成功ということになる。

## 参考資料/引用文献

ツール
- [Compiler Explorer](https://godbolt.org/)

規格書
- [Intel 64 and IA-32 Architectures Software Developer's Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- [AMD64 Architecture Programmer's Manual](https://developer.amd.com/resources/developer-guides-manuals/): AMD64 Architectureの箇所。
- [Introduction to x64 Assembly](https://www.intel.com/content/dam/develop/external/us/en/documents/introduction-to-x64-assembly-181178.pdf)
- [Documentation for binutils 2.38](https://sourceware.org/binutils/docs-2.38/)
    - [Using as](https://sourceware.org/binutils/docs-2.38/as/index.html): GNUアセンブラのリファレンス。
- [Assembler Directives](https://developer.apple.com/library/archive/documentation/DeveloperTools/Reference/Assembler/040-Assembler_Directives/asm_directives.html#//apple_ref/doc/uid/TP30000823-SW1)
- [System V ABI - OSDeV Wiki](https://wiki.osdev.org/System_V_ABI)
- [System V Application Binary Interface AMD64 Architecture Processor Supplement](https://www.uclibc.org/docs/psABI-x86_64.pdf)
- [SYSTEM V APPLICATION BINARY INTERFACE Intel386 Architecture Processor Supplement Fourth Edition](http://www.sco.com/developers/devspecs/abi386-4.pdf)
- [N1570](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf): C11言語仕様の最終草稿

記事
- [低レイヤを知りたい人のためのCコンパイラ作成入門](https://www.sigbus.info/compilerbook): インクリメンタルなCコンパイラ開発の記事
- [アセンブリに触れてみよう](https://qiita.com/kaito_tateyama/items/89272098f4b286b64115)
- [x86-64 モードのプログラミングではスタックのアライメントに気を付けよう](https://uchan.hateblo.jp/entry/2018/02/16/232029)
- [データ型のアラインメントとは何か，なぜ必要なのか？](http://www5d.biglobe.ne.jp/~noocyte/Programming/Alignment.html)
- [Cの可変長引数とABIの奇妙な関係](https://qiita.com/qnighy/items/be04cfe57f8874121e76)
- [自作Cコンパイラで（x64 ABIと戦って）セルフホストに成功した話](https://qiita.com/reki2000/items/2026b04f4eb56c8cf9cf#%E3%83%A9%E3%82%B9%E3%83%9C%E3%82%B9x64-abi-%E3%81%A8%E6%A7%8B%E9%80%A0%E4%BD%93%E3%81%AE%E5%80%A4%E6%B8%A1%E3%81%97%E3%81%A8-%E5%8F%AF%E5%A4%89%E9%95%B7%E5%BC%95%E6%95%B0)
- [C言語の関数型と関数ポインタ型の話](https://qiita.com/bellbind/items/a807e796ba80b9c94e66)
- [C で関数に * や & を付けられる件の説明: ***printf の謎](https://qiita.com/lo48576/items/92f1fc90643373d0b167)
- [セルフホスト可能なCコンパイラを書く](https://nkon.github.io/Compiler/)
- [Linuxのコンパイル結果が同等であることの確認方法](http://seigaji.info/wordpress/2015/02/15/linux_elf_binary_diff/)

書籍
- [岩波講座ソフトウェア科学5 プログラミング言語処理系](https://www.amazon.co.jp/%E5%B2%A9%E6%B3%A2%E8%AC%9B%E5%BA%A7-%E3%82%BD%E3%83%95%E3%83%88%E3%82%A6%E3%82%A7%E3%82%A2%E7%A7%91%E5%AD%A6%E3%80%88%E3%80%94%E7%92%B0%E5%A2%83%E3%80%955%E3%80%89%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0%E8%A8%80%E8%AA%9E%E5%87%A6%E7%90%86%E7%B3%BB-%E4%BD%90%E3%80%85-%E6%94%BF%E5%AD%9D/dp/4000103458): プログラミング言語処理系に関する理論と実装上の話を網羅的に解説。実装時に選択する様々なデータ構造とアルゴリズムの検討。特定の目的機械を想定していないので、コード生成の章はメモリの確保やレジスタ割り当てのアルゴリズムの解説が主となっている。

リポジトリ
- https://github.com/titech-cpp/c-compiler
- https://github.com/Imperi13/my-cc
- https://github.com/hsjoihs/c-compiler