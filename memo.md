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

## LLVMはデフォルトでは間接呼び出ししか使えない

- LLVMは関数呼び出し/関数宣言は常にCALLXnが使われる
  - TODO: ここにコードと対応するアセンブリをおく
- CALLXnは間接呼び出し
- 普通の関数呼び出し(CALLn命令)はPC相対オフセットに18ビットしか使えない
  - TODO: ここにCALLn命令のエンコードをおく
- espressif/LLVMのheadでは特定の関数属性をつけることで対応可能になっている
  - [short-callの対応コミット](https://github.com/espressif/llvm-project/commit/df27f60198198354d706eac4d7a6e588f0648db5)
  - esp-rs/rustにはまだおちてきていないので利用不可(2024/03/10現在)
