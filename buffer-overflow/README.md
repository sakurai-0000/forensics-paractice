# buffer-overflow
buffer-overflow

# 準備
## 対象プログラム
本件はC言語で記述
「A」65文字以上で壊れるプログラム
```
#include <stdio.h>
char *gets(char *);

void vulnerable(void) {
    char buf[64];
    puts("Enter:");
    gets(buf);
    printf("%s\n", buf);
}
```

コンパイル
“わざと脆弱”なバイナリを作るための設定をする。
```
gcc -g -O0 -fno-stack-protector -z execstack -no-pie bof.c -o bof
```

| オプション                   | 何をするか.             | 実質的な効果（バッファオーバーフロー実験に重要なポイント） |
| -------------------------- | --------------------- | ---------------------------------------------------------- |
| **`-g`**                   | デバッグ情報（シンボル）を埋め込む| gdb で変数名・ソース行が見える|
| **`-O0`**                  | 最適化を完全にオフ | コンパイラがコードを並べ替えないので、<br>スタックレイアウトが読みやすくなる|
| **`-fno-stack-protector`** | スタックカナリアを生成しない | 返り番地を書き換えても *\_\_stack\_chk\_fail()* が呼ばれず、<br>そのまま崩壊させられる |
| **`-z execstack`**         | スタックを「実行可能」にする| NX (Non-Executable) ガードを外し、<br>スタック上のシェルコードがそのまま実行できる|
| **`-no-pie`**              | PIE (位置独立実行ファイル) を無効化 | テキスト領域が固定アドレスでロードされる ⇒ ROP/ret2libc でアドレスを計算しやすい|
| **`bof.c`**                | 入力ソース| 脆弱な C プログラム|
| **`-o bof`**               | 出力ファイル名 | 実行ファイル `bof` を生成 |

簡単まとめると、
- デバッグしやすく (-g -O0)
- スタック破壊検知を無効化 (-fno-stack-protector)
- スタック上コードを許可 (-z execstack)
- 実行アドレスを固定 (-no-pie)

つまり
長い入力でリターンアドレスを上書き　→ シェルコード実行／ROP　が可能 

# 実行

```
$ python3 -c 'print("A"*65)' | ./bof
Enter something:
You typed: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```
うん、壊れない。。。

```
$ python3 -c 'print("A"*73)' | ./bof
Enter something:
You typed: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
zsh: done       python3 -c 'print("A"*73)' | 
zsh: bus error  ./bof
```
73文字以上で壊れる。。。
８文字分がどこかに保管されてる？？

# デバッグ

文字を入力しているvulnerableを逆アセンブリしてみる。

```
└─$ gdb ./bof
(gdb) list
2       #include <string.h>
3
4       char *gets(char *); 
5
6       /* --- intentionally vulnerable --- */
7       void vulnerable(void) {
8           char buf[64];
9
10          puts("Enter something:");
11          gets(buf);                      // <-- これが原因で BOF
(gdb) disassemble vulnerable
Dump of assembler code for function vulnerable:
   0x000000000040070c <+0>:     stp     x29, x30, [sp, #-80]!
   0x0000000000400710 <+4>:     mov     x29, sp
   0x0000000000400714 <+8>:     adrp    x0, 0x400000
   0x0000000000400718 <+12>:    add     x0, x0, #0x780
   0x000000000040071c <+16>:    bl      0x4005b0 <puts@plt>
   0x0000000000400720 <+20>:    add     x0, sp, #0x10
   0x0000000000400724 <+24>:    bl      0x4005d0 <gets@plt>
   0x0000000000400728 <+28>:    add     x0, sp, #0x10
   0x000000000040072c <+32>:    mov     x1, x0
   0x0000000000400730 <+36>:    adrp    x0, 0x400000
   0x0000000000400734 <+40>:    add     x0, x0, #0x798
   0x0000000000400738 <+44>:    bl      0x4005c0 <printf@plt>
   0x000000000040073c <+48>:    nop
   0x0000000000400740 <+52>:    ldp     x29, x30, [sp], #80
   0x0000000000400744 <+56>:    ret
End of assembler dump.
(gdb) 
```

x29 / x30 は レジスタの値。
x29はこの関数のフレーム基準アドレス
x30は戻り先アドレス。
なので基本はこに入力文字列とは無関係

```
(gdb) run < <(python3 -c 'print("D"*75)')
Starting program: /home/kali/bufferOverflow/bof < <(python3 -c 'print("D"*75)')
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/aarch64-linux-gnu/libthread_db.so.1".
Enter something:
You typed: DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDD

Program received signal SIGSEGV, Segmentation fault.
0x0000ffff00444444 in ?? ()
(gdb) info registers x30 
x30            0xffff00444444      281470686217284
(gdb) info registers x29
x29            0x4444444444444444  4919131752989213764
(gdb) 
```
x30はそのまま。
ただ、x29は0x4444444444444444 → DDDD...が格納されている。
入ってはいけないアドレスが格納される場所にinputされた文字列が格納されていることがわかる。
これはバッファーオーバーフローの証拠。