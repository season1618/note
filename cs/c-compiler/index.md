# Cコンパイラ

Linux上で動作するx86-64用のCコンパイラを作ったときのメモ。自分が実装するときに考えたことをまとめました。間違っている部分も多くあると思います。特にセルフホストの前処理の部分は自信がありません。

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

文字列をいくつかのトークンに分割する。
まず空白('\t', '\n', '\v', '\f', '\r', ' ')とコメント文を除き、トークンの判定を行う。

### トークン

トークンの分類。トークンを前方一致した時点で判断できるものとそうでないものに分ける。括弧や演算子は一致した時点で確定するので前者、keywordsは文字列が一致してかつ、その直後にアルファベット、数字、'_'以外の文字が来る必要があるので後者。punctsとkeywordsはTK_RESERVEDにまとめた。

```c tokenize.c
char *types[17] = {"extern", "const", "volatile", "static", "signed", "unsigned", "void", "_Bool", "char", "short", "int", "long", "float", "double", "struct", "union", "enum"};
char *keywords[12] = {"include", "typedef", "return", "if", "else", "switch", "case", "while", "for", "continue", "break", "sizeof"};
char *puncts[41] = {
    "+=", "-=", "*=", "/=", "%=", "||", "&&", "==", "!=", "<=", ">=", "<<", ">>", "++", "--", "->", 
    "#", "=", "?", ":", "|", "^", "&", "<", ">", "+", "-", "*", "/", "%", "&", "!", ".", ",", ";", "(", ")", "{", "}", "[", "]"
};
```

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

トークン列の判定は次のように行われる。
- 最長一致: 候補が複数ある場合、長いトークンから順に判定する。
    - `--`は`--`と判定。`- -`は`-`, `-`と分割される。
<!-- - 代替トークン: [], {}, #, ##の代わりに<::>, <%%>, %:, %:%:が使える。

```c
int a<:3:>;
a<:0:> = 12;
printf("%d\n", a[0]);
```

```
12
``` -->

### 識別子
変数名や関数名などの識別子

[N1570](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf) 6.4.2.1 General
```
identifier:
    identifier-nondigit
    identifier identifier-nondigit
    identifier digit
identifier-nondigit:
    nondigit
    universal-character-name
    other implementation-defined characters
nondigit: one of 
    _abcdefghijklm
    nopqrstuvwxyz
    ABCDEFGHIJKLM
    NOPQRSTUVWXYZ
digit: one of
    0123456789
```

- 先頭はnondigit
- ハイフンはマイナスと被るので区切り文字として不可(Lispだと使える)。

N1570 6.4.2.1 General
> There is no specific limit on the maximum length of an identifier.

- 現在の関数名を格納する`__func__`が暗黙に定義されている。

## 構文解析

トークン列を読み込み、抽象構文木(AST)を構築する。

### 構文

今回実装するCのサブセットのEBNF記法。

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

- 単独の statement として変数を宣言しても使用されないので不可。

```c
int main(){
    if(1) int a = 1;
}
```

```
error: expected expression before 'int'
    5 |     if(1) int a = 1;
      |           ^~~
```

### ノード

ノードの種類を細かく分類し、子要素を持つ。変数名や関数名以外のトークンの文字列は消去する。型のノードは作らず、修飾子ノードの属性として付与する。

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

#### 構造体と列挙型

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

### 関数定義とグローバル変数

関数名とグローバル変数名はラベルとしてアセンブリ上に残り、それらのアドレスはリンク時に決定される。一方、関数の引数はその場で宣言してローカル変数表に登録すればローカル変数と同じように扱える。従って、関数定義ノードは関数名、型、処理内容、関数全体で定義されるローカル変数全体のオフセットを持っておく必要がある。またグローバル変数ノードは名前と型、そして初期値を持っておく。

- グローバル変数は宣言が重複しても構わない(標準ライブラリで重複した宣言が存在)

関数定義とグローバル変数はトップレベルに線形に並ぶので、それらのノードのキューを作り、パーサはその先頭要素を返す。

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

### シンボルテーブル
識別子のスタックを作り、変数が登場する度に宣言済かどうか確認してpushする。

- ローカル変数、グローバル変数、タグ名の3つの表を作る。ローカル変数で名前解決できなかった場合にグローバル変数で名前解決を試みる。
- 関数定義とグローバル変数はアセンブリとして展開されるので、関数名はグローバル変数表に加える。
- 構造体と列挙型のタグは同じタグ表に加える。
- typedefで定義された型名は変数表に加える。
- 列挙型のメンバは変数表に加える。

ブロックを抜ける際にローカル変数表のスタックをpopすれば静的スコープとなる。

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

#### 制御文

`"else if"`というキーワードはなく、`"else" stmt`は`"else" "{" stmt "}"`と同じ意味となる。C言語の場合`else`の前に`if`を書く必要はあるが、`else`の後に`if`を書く必要はない。

```c
int a = 0;
if(a != 0){
    printf("%d\n", a);
}else while(a < 5){
    printf("%d\n", a);
    a++;
}
```

```
0
1
2
3
4
```

- Pythonでは`while`や`for`の後にも`else`を書くことができる。

[Python documentation 8.1. The if statement](https://docs.python.org/3/reference/compound_stmts.html#the-if-statement)
```
if_stmt ::=  "if" assignment_expression ":" suite
             ("elif" assignment_expression ":" suite)*
             ["else" ":" suite]
while_stmt ::=  "while" assignment_expression ":" suite
                ["else" ":" suite]
for_stmt ::=  "for" target_list "in" expression_list ":" suite
              ["else" ":" suite]
```

- 宙ぶらりんelse問題(dangling else)

最も近くにある対応していない`if`と結合する。

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

制御文のパースは各要素を読んで代入するだけ。

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

`switch`, `while`, `for`文についても基本的に同様。

- `switch`文のスタックを持っておき、それにpushする。
- `case`文はルート方向に最も近い`switch`文を持っておく。
- `for`文の括弧内で宣言された変数を表から削除し、スコープから除く。

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

#### 二項演算

演算は基本的に演算子ごとにノードを用意する。

足し算は実際には次のように再帰的に定義されている。

[N1570](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf)6.5.6 Additive operators
```
additive-expression:
    multiplicative-expression
    additive-expression + multiplicative-expression
    additive-expression - multiplicative-expression
```

再帰的な書き方は演算子の結合性を表現するのに都合が良いが、プログラムで左から構文解析すると、この場合左再帰なので無限ループとなる。
そこで以下のように等価な文法に書き換える。

```
add = mul ("+" mul | "-" mul)*
```

これを実装すると次のようになる。

```c:parse.c
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

#### sizeof

`sizeof`演算はコンパイル時に計算できるのでノードは用意しなくても良い。

```c parse.c
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

識別子を読み込んだらシンボルテーブルにアクセスして名前解決を試みる。ローカル変数なら型とオフセット、グローバル変数なら型と名前をコピーする。

## プログラム実行の流れ

人間が書いたソースコードはプリプロセスを経た後、コンパイラによってアセンブリコードへと変換され、更にアセンブラによってオブジェクトコードに変換される。複数のオブジェクトファイルはリンカによって結合され、最終的な実行ファイルが生成される。

プログラムを実行する際、OSのプログラムローダが実行ファイルの内容をメモリ上にコピーする。CPUはプログラムカウンタという領域に現在実行中の命令のアドレスを保持しており、そのアドレスから順次命令を読みだして実行する。CPUはレジスタと呼ばれる記憶領域を持っており、メモリから読んだデータをレジスタにコピーして処理を行う。レジスタはCPU内部に存在するので、メモリに比べてアクセスが速い。

UNIXシステムにおけるオブジェクトファイルと実行ファイルには同じフォーマットが用いられる。過去にはa.outやCOFF(Common Object File Format)などのファイル形式が使われていたが、現在ではELF(Executable and Linkable Format)が広く採用されている。

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

| bit | ニーモニック | 説明 | 用途 | 1となる条件 |
|--|--|--|--|--|
| 11 | OF |        Overflow Flag | 符号付き整数演算の結果の最上位bitが本来と異なる |
| 10 | DF |       Direction Flag | 文字列操作のデータポインタが減る向き |
|  7 | SF |            Sign Flag | 算術演算の結果が負 |
|  6 | ZF |            Zero Flag | 算術演算の結果が0 |
|  4 | AF | Auxiliary Carry Flag |  |
|  2 | PF |          Parity Flag | 演算の結果の最下位byte |
|  0 | CF |           Carry Flag | 加算/減算命令で最上位bitで繰り上がり/繰り下がりが発生 |

DFはcontrol flag、他の6個はstatus flagと呼ばれている。

128bitSSE(Streaming SIMD Extended)レジスタ
80bit x87浮動小数点レジスタ

SSEレジスタはvector registerとも呼ばれる。

### メモリモデル
メモリは1byteごとに並んでいる。メモリには大きく四つの領域がある。プログラムそのものとプログラムが扱うデータはどちらもメモリに入っている。

AMD64はリトルエンディアン(低ビットが低アドレス)なので、例えば32bit以下の64bit整数を`DWORD PTR [rax]`で読んでも`QWORD PTR [rax]`で読んでも全く問題ない。

64bitモードでは、デフォルトのオペランドサイズは基本的に4byteであり、REXプリフィックスの付与された命令では8byteである。ただし、暗黙にスタックポインタを参照する命令についてはデフォルトで8byteオペランドサイズを採用する。

[AMD64 Architecture Programmer's Manual Volume 1: Application Programming](https://www.amd.com/system/files/TechDocs/24592.pdf) 3.2.3.1 Default Operand Size
> There are several exceptions to the 32-bit operand-size default in 64-bit mode, including near branches
and instructions that implicitly reference the RSP stack pointer. For example, the near CALL, near
JMP, Jcc, LOOPcc, POP, and PUSH instructions all default to a 64-bit operand size in 64-bit mode.
Such instructions do not need a REX prefix for the 64-bit operand size. For details, see “GeneralPurpose Instructions in 64-Bit Mode” in Volume 3. 

### アドレッシング
メモリにアクセスする方法をアドレシングという。実効アドレスの生成には以下の5つの方法がある。

- 完全なアドレス: Base + Displacement(Offset)
- RIP相対アドレシング: IP(PC) + Displacement
- インデックス付きレジスタ間接: Base + Scale * Index + Displacement, Scale = 1, 2, 4, 8
- スタックアドレス
- string address

### 命令セット
命令は一般に<mnemonic> <source or destination> <source>の形で書かれる。
64bitモードではデフォルトのアドレスサイズは64bitである。

#### データ転送命令
レジスタ/メモリからレジスタ/メモリへデータを移動する。ただしメモリからメモリへの移動はできない。第二オペランドとして即時定数(immediate constant)を用いることができる。
- mov
- movzx: ゼロ拡張(zero extention)。ソースより大きいレジスタに転送する際、上位bitをゼロ埋めする。
- movsx: 符号拡張(sign extention)。ソースより大きいレジスタに転送する際、符号を考慮して上位bitに拡張する。

#### スタック操作
- push: スタックポインタをデータサイズ分減らし、スタックトップにデータをコピーする。
- pop: スタックトップからデータをコピーし、スタックポインタをデータサイズ分増やす。

#### 有効アドレス
- lea: load effective addressの略。

#### 算術演算
- add a b: a += b
- sub a b: a -= b
- imul a b: a *= bの符号あり乗算。
- idiv a: rax / aの符号あり除算。商と剰余をそれぞれrax, rdxに格納する。
- neg: 2の補数を求める。
- dec: デクリメント。CFは影響しない。
- inc: インクリメント。CFは影響しない。

#### 比較演算
- cmp a b: a - bを計算し、結果に従ってフラグレジスタを変化させる。

直前のcmp命令の結果を8bitレジスタに格納。

#### 条件命令
条件に従い、byteオペランドに0, 1を格納。
- sete c: a == b ならcに1を格納。
- setne c: a != b ならcに1を格納。
- setl c: a < b ならcに1を格納。
- setle c: a <= b ならcに1を格納。
- \>, >=はない。

#### 制御転送
- je: 条件ジャンプ
- call: スタックにリターンアドレスを積み、関数アドレスへジャンプ。
- ret: スタックからアドレスを一つポップしてジャンプ。

制御転送命令はスタックアラインメントが適切でない場合、著しくパフォーマンスが落ちる。

[AMD64 Architecture Programmer's Manual Volume 1: Application Programming](https://www.amd.com/system/files/TechDocs/24592.pdf) 3.7.3.1 Stack Alignment
> Control-transfer performance can degrade significantly when the stack pointer is not aligned properly.
Stack pointers should be word aligned in 16-bit segments, doubleword aligned in 32-bit segments, and
quadword aligned in 64-bit mode. 

[Intel® 64 and IA-32 Architectures Software Developer’s Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) Volume1: Basic Architecture 6.2.2 Stack Alignment
> The stack pointer for a stack segment should be aligned on 16-bit (word) or 32-bit (double-word) boundaries,
depending on the width of the stack segment. The D flag in the segment descriptor for the current code segment
sets the stack-segment width (see “Segment Descriptors” in Chapter 3, “Protected-Mode Memory Management,” of
the Intel® 64 and IA-32 Architectures Software Developer’s Manual, Volume 3A). The PUSH and POP instructions
use the D flag to determine how much to decrement or increment the stack pointer on a push or pop operation,
respectively. When the stack width is 16 bits, the stack pointer is incremented or decremented in 16-bit increments;
when the width is 32 bits, the stack pointer is incremented or decremented in 32-bit increments. Pushing a 16-bit
value onto a 32-bit wide stack can result in stack misaligned (that is, the stack pointer is not aligned on a double word boundary). One exception to this rule is when the contents of a segment register (a 16-bit segment selector)
are pushed onto a 32-bit wide stack. Here, the processor automatically aligns the stack pointer to the next 32-bit
boundary
> The processor does not check stack pointer alignment. It is the responsibility of the programs, tasks, and system
procedures running on the processor to maintain proper alignment of stack pointers. Misaligning a stack pointer
can cause serious performance degradation and in some instances program failures.

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

## X86アセンブリ言語
gccなどの処理系では、アセンブラとして主にGNUアセンブラ(GNU assembler, gas)が用いられており、GNUアセンブラが出力するアセンブリ言語もGNUアセンブラの仕様書で定義されている。アセンブリ言語の文法には一部アーキテクチャに依存する部分がある。

`.text`, `.data`, `.bss`ディレクティブを付けることで、実行ファイル上でのセクションを指定することができる。

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

GNUアセンブラではデフォルトではAT&T記法を用いるが、2.10からは`.intel_syntax`ディレクティブを付けることでIntel記法が使えるようになった。ここではIntel記法で説明する。

### メモリ参照
- 間接メモリ参照: size [base + index * scale + displacement]
- RIPアドレッシング: [rip + offset]

## System V ABI(Application Binary Interface)

アセンブリの書き方はOSによって規約が定められており、これをABIという。それによってコンパイラの挙動も合わせる必要がある。Linux上でx86-64を実行するときはSystem V ABIというものが使われる。System V ABIには、レジスタの用途、関数の呼び出し規約、オブジェクトファイルおよび実行ファイルのフォーマットなどが定めらている。ベースとなるドキュメントとプラットフォーム依存の補遺からなる。

<!-- x86
- cdecl
- stdcall

x86-64
- System V ABI: Unix, macOS
- Microsoft ABI: Windows -->

<!-- データモデル
- LP64(System V ABI)
    - 32bit: int
    - 64bit: long, pointer
- LLP(Microsoft)
    - 32bit: int, long
    - 64bit: long long, pointer -->

### レジスタ
System V ABIでは汎用レジスタに以下の用途が指定されている。

64bit汎用レジスタ
| 64bitレジスタ | 32bitレジスタ | 16bitレジスタ | 8bitレジスタ | 用途 |
|--|--|--|--|--|
| RSP | ESP |  SP | SPL | スタックポインタ |
| RBP | EBP |  BP | BPL | ベースポインタ |
| RAX | EAX |  AX |  AL | 返り値 |
| RBX | EBX |  BX |  BL ||
| RDI | EDI |  DI | DIL | 第一引数 |
| RSI | ESI |  SI | SIL | 第二引数 |
| RDX | EDX |  DX |  DL | 第三引数 |
| RCX | ECX |  CX |  CL | 第四引数 |
| R8  | R8D | R8W | R8B | 第五引数 |
| R9  | R9D | R9W | R9B | 第六引数 |
| R10 | R10D | R10W | R10B ||
| R11 | R11D | R11W | R11B ||
| R12 | R12D | R12W | R12B ||
| R13 | R13D | R13W | R13B ||
| R14 | R14D | R14W | R14B ||
| R15 | R15D | R15W | R15B ||

### 構造体のデータアラインメント

多くのCPUではメモリに格納するデータのサイズによって境界が定まっており、アラインメントと呼ばれる。例えば4バイト境界なら4の倍数のアドレスから順にデータを読み出す必要がある。

x86-64アーキテクチャでは基本的にデータ型のアラインメントを揃える必要はないが、パフォーマンスの観点から揃えることを推奨している。

[AMD64 Architecture Programmer's Manual Volume 1: Application Programming](https://www.amd.com/system/files/TechDocs/24592.pdf) 3.2.5 Data Alignment
> The AMD64 architecture does not impose data-alignment requirements for accessing data in memory.
However, depending on the location of the misaligned operand with respect to the width of the data bus and other aspects of the hardware implementation (such as store-to-load forwarding mechanisms),
a misaligned memory access can require more bus cycles than an aligned access. For maximum
performance, avoid misaligned memory accesses. 

[Intel® 64 and IA-32 Architectures Software Developer’s Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) Volume1: Basic Architecture 4.1.1 Alignment of Words, Doublewords, Quadwords, and Double Quadwords
> Words, doublewords, and quadwords do not need to be aligned in memory on natural boundaries. The natural
boundaries for words, double words, and quadwords are even-numbered addresses, addresses evenly divisible by
four, and addresses evenly divisible by eight, respectively. However, to improve the performance of programs, data structures (especially stacks) should be aligned on natural boundaries whenever possible.

[データ型のアラインメントとは何か，なぜ必要なのか？](http://www5d.biglobe.ne.jp/~noocyte/Programming/Alignment.html)
１．CPU に関する基礎知識
> アラインメントは CPU のハードウェアに起因する問題であり， データのメモリアドレスに関する制約である．

３.２ アラインメントに厳格な CPU の場合 (alignment-strict processors)
> x86 以外の多くの CPU (特に RISC) は上記のようには動作せず，エラーとして処理する．

４．アラインメントの基礎知識のまとめ
> アラインメントに寛容な CPU (x86 など) では， アラインメントを守らなくても正常に動作するが， そのデータのメモリアクセスが遅くなる．

ABIでは構造体のデータアラインメントが定義されている。
構造体のアラインメントはメンバのアラインメントの最小公倍数となる。データ型は全て2の累乗なので、結果それらの最大値となる。

[System V Application Binary Interface AMD64 Architecture Processor Supplement](https://www.uclibc.org/docs/psABI-x86_64.pdf) 3.1.2 Data Representation
> Structures and unions assume the alignment of their most strictly aligned component. Each member is assigned to the lowest available offset with the appropriate alignment. The size of any object is always a multiple of the object‘s alignment.
> An array uses the same alignment as its elements, except that a local or global array variable of length at least 16 bytes or a C99 variable-length array variable always has alignment of at least 16 bytes.4
> Structure and union objects can require padding to meet size and alignment constraints. The contents of any padding is undefined.

### 関数呼び出し
分割コンパイルする際、別のコンパイラでコンパイルしたファイルの関数を用いるためには、関数呼び出し時の引数や返り値を格納するレジスタ/スタックを揃えておく必要がある。最も単純なやり方は引数を全てスタックに格納する方法だが、アクセスの高速なレジスタを用いるために複雑な規約が定められている。

レジスタ
- rbp, rbx, r12-r15は関数呼び出しの前後で値が保存される必要がある。
-  DFは関数の開始と終了時に順方向にリセットする。

スタックフレーム
| アドレス | 値 |
|--|--|
| rsp + 32 | 引数 |
| rsp + 16 | 引数 |
| rsp + 8  | リターンアドレス |
| rsp      | rbpの値 |

- スタックアラインメント: callの直前、rspは16の倍数になっている必要がある。

引数渡し
- 整数はrdi, rsi, rdx, rcx, r8, r9に左から順に格納し、それ以外はスタックに積む(今回は整数しか使っていないので)。

### ELFファイル
ELFは少なくとも三つのセグメントに分かれており、text, data, bssの順に並んでいる。bssセグメントには0に初期化されるデータが格納されている。
<!-- > The advantage in using the bss segment for storage that starts off empty is that the initialization information need not be stored in the output file. -->

## コード生成

構文解析によって得たASTを深さ優先探索する。式(expression)や`return`はオペランドをスタックからpopし、返り値をpushするようにしておく(連続する操作でpop, push命令が連続することになるが、そういった最適化は後回しにする)。それ以外の結果を返さないノードではスタックを空にして収支を合わせるようにする。
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
次の関数呼び出しに備えて、引数の値をレジスタまたはスタックからrbp直下のスタック領域に書き出す。第一引数、第二引数を[rbp-8], [rbp-16],...に格納する。

```c codegen.c
// 略

int num_param_int = 0;
for(symb *param = nd->ty->head; param; param = param->next){
    if(num_param_int < 6){
        printf("    mov rax, rbp\n");
        printf("    sub rax, %d\n", param->offset);
        mov_memory_from_register(rax_, int_arg_reg[num_param_int], param->ty);
    }else{
        printf("    mov rax, rbp\n");
        printf("    sub rax, %d\n", param->offset);
        printf("    mov rbx, rbp\n");
        printf("    add rbx, %d\n", 8 * (num_param_int - 4));
        printf("    mov rbx, [rbx]\n");
        mov_memory_from_register(rax_, rbx_, param->ty);
    }
    num_param_int++;
}

gen_stmt(fn->stmt);

// 略
```

### 文(Statement)

ラベルを貼ってジャンプ命令で遷移する。

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

他の制御文も同様。

### 式(Expression)
#### 左辺値と右辺値

- 配列要素は順番に格納され、大きさの情報は実行時に失われる。
- 配列はその先頭要素のポインタに暗黙にキャストされるが、ポインタの実体があるわけではない。
- 配列のアドレスは先頭要素のアドレスと等しい。

- 構造体メンバは順番に格納され、それぞれのアドレスを保持する。
- 構造体のアドレスは先頭要素のアドレスと等しい。

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
ラベルを貼って短絡評価する。

```c codegen.c
case ND_LOG_OR:{
    int label_end = label_num;
    label_num++;

    gen_expr(nd->op1);
    printf("    pop rax\n");
    printf("    cmp rax, 0\n");
    printf("    jne .L%d\n", label_end);

    gen_expr(nd->op2);
    printf("    pop rax\n");

    printf(".L%d:\n", label_end);
    printf("    push rax\n");
    return;
}
```

#### 関数呼び出し
スタックのアラインメントを調整した上で、引数をレジスタまたはスタックに格納して`call`命令を行う。

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

しかしgccでリンクした場合、アセンブリが同一でも実効ファイルが同じとは限らない。
[Linuxのコンパイル結果が同等であることの確認方法](http://seigaji.info/wordpress/2015/02/15/linux_elf_binary_diff/)
> まず差分が発生する理由として考えられるのがコンパイラがビルドする際にリンカが使うためのシンボル情報を付加しているというのは常識。ということでstripコマンドを使ってみることにしました。

リンカによって付加されるシンボル情報が異なるらしい。つまり、
```
> strip -s stage2
> strip -s stage3
> diff stage2 stage3
```
で何も出なければセルフホスト成功ということになる。

## 嵌った点
コンパイラを作っていると普段は出会えないエラーに遭遇することになります。

### 構造体と変数名に同じ名前を使うとキャストできない
`(my_struct*)`を外すと通る。
```c
my_struct *new_my_struct(int val){
    my_struct *my_struct = (my_struct*)calloc(1, sizeof(my_struct));
    my_struct->val = val;
    return my_struct;
}
```

```
error: expected expression before ')' token
my_struct *my_struct = (my_struct*)calloc(1, sizeof(my_struct));
```

あとメンバのアクセスでバグるぽい。(良く分からない)

```
sysmalloc: Assertion `(old_top == initial_top (av) && old_size == 0) || ((unsigned long) (old_size) >= MINSIZE && prev_inuse (old_top) && ((unsigned long) old_end & (pagesize - 1)) == 0)' failed. Aborted
```

### `node *nd = calloc(1, sizeof(node*))`が暗黙キャストされてメモリが確保されなかった
気を付けましょう。

### 関数呼び出しの引数内で計算があるとバグる
関数呼び出しの引数内で割り算の式があった。
割り算は以下のように行う。
```
cqo
idiv rdi
```
`cqo`は`rax`を`rdx:rax`に拡張する命令。`idiv rdi`は`rdx:rax`と`rdi`の割り算を計算する。後ろの引数から評価する際、`cqo`命令が既に第三引数を格納していた`rdx`レジスタを上書きしてしまったため正常に動作しなかった。
解決策としては、引数を右から評価する際に一旦全てをスタックに積んでから、最初の引数をレジスタに移動させる。

各ノードの生成されたコードで、最初にスタックからpopして最後にpushするが、収支が合わず余計にpopされてしまっていた。elseなしのif文では文があるかないかなので、ジャンプした先で余計にpopされていた。

### グローバル変数名にレジスタ名を使うとアセンブラに怒られる
gcc + intel syntaxだとグローバル変数名にレジスタ名を使うと`rax[rip]`が出力されて不可となる。
![](./images/global_register.png)

### NULLをmemcmpに渡すとSegmentation fault
文字列の一致判定を自作コンパイラでコンパイルするときに以下のようなコードで発生。
```c
if(a->len == b-len && memcmp(a->name, b->name, b->len) == 0)
```

自作コンパイラでは論理演算の短絡評価をしていなかったので、`a->name`が`NULL`の時に第二オペランドが評価されてSegmentation faultした。

## 参考資料/引用文献

ツール
- [Wandbox](https://wandbox.org/)
- [Compiler Explorer](https://godbolt.org/)

規格書
- [AMD64 Architecture Programmer's Manual](https://developer.amd.com/resources/developer-guides-manuals/): AMD64 Architectureの箇所。特に[Volume 1: Application Programming](https://www.amd.com/system/files/TechDocs/24592.pdf)の1 Overview of the AMD64 Architecture, 2 Memory Model, 3 General Purpose Programming, 6 x87 Floating-Point Programming
- [Intel 64 and IA-32 Architectures Software Developer's Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html): Volume1 4.1.1 Alignment of Words, Doublewords, Quadwords, and Double Quadwords
- [Introduction to x64 Assembly](https://www.intel.com/content/dam/develop/external/us/en/documents/introduction-to-x64-assembly-181178.pdf)
- [Documentation for binutils 2.38](https://sourceware.org/binutils/docs-2.38/)
    - [Using as](https://sourceware.org/binutils/docs-2.38/as/index.html): GNUアセンブラのリファレンス。7 Assembler Directives。
        - [9.16.3.1 AT&T Syntax versus Intel Syntax](https://sourceware.org/binutils/docs-2.38/as/i386_002dVariations.html#i386_002dVariations)
        - [9.16.7 Memory References](https://sourceware.org/binutils/docs-2.38/as/i386_002dMemory.html)
- [UNIX Assembler Reference Manual](https://www.tom-yam.or.jp/2238/ref/as.pdf)
- [Assembler Directives](https://developer.apple.com/library/archive/documentation/DeveloperTools/Reference/Assembler/040-Assembler_Directives/asm_directives.html#//apple_ref/doc/uid/TP30000823-SW1)
- [System V ABI - OSDeV Wiki](https://wiki.osdev.org/System_V_ABI)
- [System V Application Binary Interface AMD64 Architecture Processor Supplement](https://www.uclibc.org/docs/psABI-x86_64.pdf): 3.1 Machine Interface, 3.2 Function Calling Sequence, 3.5.7 Variable Argument List
- [SYSTEM V APPLICATION BINARY INTERFACE Intel386 Architecture Processor Supplement Fourth Edition](http://www.sco.com/developers/devspecs/abi386-4.pdf)
<!-- (https://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf) -->
- [N1570](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf): C11言語仕様の最終草稿
- [The Python Language Reference](https://docs.python.org/3/reference/compound_stmts.html): Python3のリファレンス

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