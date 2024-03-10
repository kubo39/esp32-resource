# メモ

あらすじ: [A:Mac合宿2024](https://amac.connpass.com/event/308090/)に向かうことになった俺達はひょんなことから発表することに。
わたしどうなっちゃうの〜〜？

## ABI

- 二種類あるがWindowed register ABIしか取り扱わない
- レジスタは32bit
- 引数用のレジスタはa2-a7の6個
- 返り値用のレジスタはa2-a5の4個
- a0,a1は特別
  - a0: リターンアドレス
  - a1: スタックポインタ

## Windowed register

CALLn/CALLXnの呼び出し規約のcaller(呼び出し元)/callee(呼び出し先)対応は特殊で、

- callerのa_n GPRがcalleeのa0に対応
- callerのa_n+1 GPRがcalleeのa1に対応
- さらにa_n+2,a_n+3も同様に…

というようなマッピングになっている。

```c
/*
* int bar(int x, int y);  // callee
*
* void func(void)  // caller
* {
* int foo;
* foo = bar(x, y);
* ...
* }
*/
func:
...
mov a10, x // a10 is bar’s a2 (第一引数)
mov a11, y // a11 is bar’s a3 (第二引数)
call8 bar
mov foo, a10 // a10 is bar’s a2 (return value)
```

このcaller/calleeのレジスタの対応をやってくれるのがentry命令とretw命令。

## LLVMはデフォルトでは間接呼び出ししか使えない

LLVMは関数呼び出し/関数宣言は常にCALLXnが使われる。

(TODO: ここにコードと対応するアセンブリをおく)

- CALLXnは間接呼び出し
- 普通の関数呼び出し(CALLn命令)はPC相対オフセットに18ビットしか使えない
  - TODO: ここにCALLn命令のエンコードをおく

espressif/LLVMのheadでは特定の関数属性をつけることで対応可能になっている([short-callの対応コミット](https://github.com/espressif/llvm-project/commit/df27f60198198354d706eac4d7a6e588f0648db5))。

ただしesp-rs/rustにはまだおちてきていないので利用不可(2024/03/10現在)。
