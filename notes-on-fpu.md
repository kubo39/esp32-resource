# Rustはesp32s3でちゃんとFPUを使ってくれるのか？

Rustはesp32s3でちゃんとFloating Point Coprocessorを扱ってくれるのか？(正確にいうと、デフォルトで除算やsqrtなどの実装がsingle fp拡張に最適化された実装になってくれるのか)というのが気になる。

## esp-rs/rustのバイナリはどう転がってくるか

まずRustコンパイラを入れる最も一般的な方法は[espup][espup]を利用する方法であるので、そこからみていく。

```console
$ cargo install espup
$ espup -V
espup 0.11.0
$ espup install
```

現在最新の安定板は0.11.0なので、こちらのコードをみていく。

Rustコンパイラのダウンロードは[rustbuild][rustbuild]レポジトリの最新のリリースを常に見に行くようになっている。執筆時点で1.76.0.1となっていたが、これは
実際のespupで入れた内容と合致している。

```console
$ . $HOME/export-esp.sh
$ rustup run esp rustc -Vv
rustc 1.76.0-nightly (88269fa9e 2024-02-09) (1.76.0.1)
binary: rustc
commit-hash: 88269fa9ed1d862991d52315f6d76d064407a5c0
commit-date: 2024-02-09
host: x86_64-unknown-linux-gnu
release: 1.76.0-nightly
LLVM version: 17.0.1
```

ではこのRustコンパイラが利用しているLLVMがどこかというとこれは[rustcのsubmoduleで指定](https://github.com/esp-rs/rust/blob/esp-1.76.0.1/.gitmodules#L33-L37)されている。

現在利用されているLLVMは[Xtensa_Release_17.0.1][llvm_xtensa_release_17.0.1]ブランチで、これ機能追加が活発に行われているブランチなので実質的にupstreamとみてよさそうだ。

ついでにespupでインストールされるGCCツールチェーンもみていく。GCC(リンカなどで使う)やbinutils(objdumpとかreadelfとか)は[espressif/crosstool-NGの特定のバージョン](https://github.com/esp-rs/espup/blob/v0.11.0/src/toolchain/gcc.rs#L18-L19)を利用してビルドしたものを利用している。

[crosstool-NGレポジトリ](https://github.com/espressif/crosstool-NG/tree/esp-13.2.0_20230928)内でどのGCCおよびbinutilsのバージョンを指定しているかというと、めちゃくちゃわかりにくいのだが[samplesディレクトリ内のcrosstool.config](https://github.com/espressif/crosstool-NG/blob/esp-13.2.0_20230928/samples/xtensa-esp-elf/crosstool.config)に記載されている。

```config
(...)
CT_GCC_DEVEL_URL="https://github.com/espressif/gcc.git"
CT_GCC_DEVEL_BRANCH="esp-13.2.0_20230928"
(...)
CT_BINUTILS_DEVEL_URL="https://github.com/espressif/binutils-gdb.git"
CT_BINUTILS_DEVEL_BRANCH="esp-2.41.0_20230928"
```

こちらも手元の環境と合致していることが確認できる。

```console
$ xtensa-esp32s3-elf-gcc --version| head -1
xtensa-esp-elf-gcc (crosstool-NG esp-13.2.0_20230928) 13.2.0
```

これで[espupが入れるGCCのバージョンタグ](https://github.com/espressif/gcc/tree/esp-13.2.0_20230928)がわかった。

それではこれが本当に利用されているかesp-idfプロジェクトを作ってためしてみよう。

```console
$ cargo generate esp-rs/esp-idf-template cargo
⚠️   Favorite `esp-rs/esp-idf-template` not found in config, using it as a git repository: https://github.com/esp-rs/esp-idf-template.git
🤷   Project Name: rust-esp32s3-example
🔧   Destination: /home/kubo39/dev/espressif/rust-esp32s3-example ...
🔧   project-name: rust-esp32s3-example ...
🔧   Generating template ...
✔ 🤷   Which MCU to target? · esp32s3
✔ 🤷   Configure advanced template options? · false
🔧   Moving generated files into: `/home/kubo39/dev/espressif/rust-esp32s3-example`...
🔧   Initializing a fresh Git repository
✨   Done! New project created /home/kubo39/dev/espressif/rust-esp32s3-example
```

しかしこれはちょっとGCCのバージョンがずれてしまっている。

```console
$ cargo build
(...)
$ xtensa-esp32s3-elf-readelf -p .comment target/xtensa-esp32s3-espidf/debug/rust-esp32s3-example

String dump of section '.comment':
  [     0]  GCC: (crosstool-NG esp-12.2.0_20230208) 12.2.0
  [    2f]  rustc version 1.76.0-nightly (88269fa9e 2024-02-09) (1.76.0.1)
  [    6e]  GCC: (crosstool-NG esp-2021r1) 8.4.0
```

これは[esp-idf v5.1.3が指定しているtools.json](https://github.com/espressif/esp-idf/blob/v5.1.3/tools/tools.json#L326)のバージョンが古いためと思われる。
embuildはプロジェクト生成時に指定されたesp-idfのバージョンを持ってきてツールとともに展開するようになっている。

とはいえxtensa関連のコードはいずれのバージョンでも大して変わりがないのでそれほど気にしなくても問題なさそうだ。
気になる人はldproxyの引数で任意のリンカへの切り替えが可能であるので新しいバージョンに切り替えて利用するのがよいだろう。

## sqrtfの実装

ここからRustおよびLLVMが実際にどのように浮動小数点数を使った処理をコンパイルしていくのか？というのをsqrtf(32ビットの浮動小数点数のsqrtルーチン)を例にみていく。

まずRustのコンパイラフロントエンドはXtensaバックエンドを含めだいたいの場合は`llvm.sqrt.f32`というLLVM intrinsicに変換する。
なのでXtensaバックエンドがこのintrinsicをどのようにアセンブリに落とすか？という部分に注目する。

以下のようなLLVM IRが入力としてあった場合:

```ll
define float @sqrt_f32(float %a) nounwind {
  %1 = call float @llvm.sqrt.f32(float %a)
  ret float %1
}
```

LLVMは以下のようなアセンブリへと変換する。

```asm
   .literal .LCPI0_0, sqrtf
sqrt_f32:
    # %bb.0:
	entry	a1, 32
	l32r	a8, .LCPI0_0
	mov.n	a10, a2
	callx8	a8
	mov.n	a2, a10
	retw.n
```

このxtensaアセンブリを少し簡単にして説明を足すと以下のようになる。

```asm
	entry	a1, 32     # スタックフレーム確保とWindowed register ABI
	                   #  caller/calleeレジスタの対応
	l32r	a8, sqrtf  # a8レジスタにsqrtfシンボルをロード
	mov.n	a10, a2    # sqrt_f32関数の第一引数をa10レジスタに移動
	                   # (a10レジスタはsqrtf関数の第一引数となる)
	callx8	a8         # sqrtf関数を呼び出す
	mov.n	a2, a10    # a10レジスタの内容を返り値用のレジスタa2に移動
	                   # (a10レジスタにはsqrtf関数の返り値がある)
	retw.n             # 関数からリターン
```

少しだけ補足すると、esp32はxtensaのWindowed register ABIを利用しているため、

- 引数に使えるレジスタはa2-a7の6個
- 返り値に使えるレジスタはa2-a5の4個
- caller/calleeのレジスタの対応にルールがある
  - 上の例でいうと、callx8を使った場合にcallerのa10レジスタがcalleeのa2レジスタに対応する
  - entry命令はスタックフレームの確保と同時にレジスタの対応を行う
  - retw/retw.n命令はこのABIに対応したret命令

といったルールがあるが、ここで大事なのはLLVMはこの時点では存在していないsqrtf関数を呼び出そうとしているという点になる。

このsqrtf関数はどこにあるのか？

ここでRustコンパイラがリンカにgccを使っている(esp32s3の場合はxtensa-esp32s3-elf-gcc)ことを思い出してほしい。このリンカ(正確にはGCCが使うldだが)がリンクするオブジェクトファイルが別に存在しているはずだ。

この関数定義は[newlib libc側](https://github.com/espressif/newlib-esp32/blob/esp-4.3.0_20230928/newlib/libm/machine/xtensa/ef_sqrt.c)にあり、`XCHAL_HAVE_FP_SQRT`マクロが定義されている場合はGCCのビルトイン関数`__ieee754_sqrtf`関数を利用する旨が書かれている。

- [newlib/libm/machine/xtensa/Makefile.inc](https://github.com/espressif/newlib-esp32/blob/esp-4.3.0_20230928/newlib/libm/machine/xtensa/Makefile.inc)

```makefile
%C%_src = \
	%D%/feclearexcept.c %D%/fegetenv.c %D%/fegetexcept.c %D%/fegetexceptflag.c \
	%D%/fegetround.c %D%/feholdexcept.c %D%/feraiseexcept.c %D%/fetestexcept.c \
	%D%/feupdateenv.c

if XTENSA_XCHAL_HAVE_FP_SQRT
%C%_src += \
	%D%/ef_sqrt.c
endif

libm_a_CFLAGS_%C% = -D_LIBM
libm_a_SOURCES += $(%C%_src)
```

- [newlib/libm/machine/xtensa/ef_sqrt.c](https://github.com/espressif/newlib-esp32/blob/esp-4.3.0_20230928/newlib/libm/machine/xtensa/ef_sqrt.c)

```c
#include <xtensa/config/core-isa.h>

#if !XCHAL_HAVE_FP_SQRT
#error "__ieee754_sqrtf form common libm must be used"
#else
/* Built-in GCC __ieee754_sqrtf must be used */
#endif
```

[libgccのsqrtfルーチン](https://github.com/espressif/gcc/blob/esp-13.2.0_20230928/libgcc/config/xtensa/ieee754-sf.S#L1827-L1874)を有効にするマクロ定義はcrosstool-NGが参照している[xtensa-overlaysレポジトリの設定](https://github.com/espressif/xtensa-overlays/blob/dd1cf19f6eb327a9db51043439974a6de13f5c7f/xtensa_esp32s3/gcc/include/xtensa-config.h#L101)で定義されている。

```c
#undef XCHAL_HAVE_FP_SQRT
#define XCHAL_HAVE_FP_SQRT		1
```

sqrtfルーチンの内容をこちらにも記載する。

```asm
	/* Square root */

	.align	4
	.global	__ieee754_sqrtf
	.type	__ieee754_sqrtf, @function
__ieee754_sqrtf:
	leaf_entry	sp, 16

	wfr		f1, a2

	sqrt0.s		f2, f1
	const.s		f3, 0
	maddn.s		f3, f2, f2
	nexp01.s	f4, f1
	const.s		f0, 3
	addexp.s	f4, f0
	maddn.s		f0, f3, f4
	nexp01.s	f3, f1
	neg.s		f5, f3
	maddn.s		f2, f0, f2
	const.s		f0, 0
	const.s		f6, 0
	const.s		f7, 0
	maddn.s		f0, f5, f2
	maddn.s		f6, f2, f4
	const.s		f4, 3
	maddn.s		f7, f4, f2
	maddn.s		f3, f0, f0
	maddn.s		f4, f6, f2
	neg.s		f2, f7
	maddn.s		f0, f3, f2
	maddn.s		f7, f4, f7
	mksadj.s	f2, f1
	nexp01.s	f1, f1
	maddn.s		f1, f0, f0
	neg.s		f3, f7
	addexpm.s	f0, f2
	addexp.s	f3, f2
	divn.s		f0, f1, f3

	rfr		a2, f0

	leaf_return
```

ちゃんとsingle fp拡張命令とFP用のレジスタが利用されており、引数として渡ってくるa2レジスタと返り値として渡す必要があるa2レジスタへの操作以外はGPRs(General Purpose Registers)を利用しないよう注意深く実装されている。

## 実際に

論より証拠、ということで実際のバイナリを見るのが結局一番である。

テンプレートで生成したコードにsqrtメソッドを使う処理を追加する。

```rust
fn main() {
    // It is necessary to call this function once. Otherwise some patches to the runtime
    // implemented by esp-idf-sys might not link properly. See https://github.com/esp-rs/esp-idf-template/issues/71
    esp_idf_svc::sys::link_patches();

    // Bind the log crate to the ESP Logging facilities
    esp_idf_svc::log::EspLogger::initialize_default();

    log::info!("Hello, world!");

    // 以下二行を追加
    let ans = (5.0_f32).sqrt();
    log::info!("(5.0).sqrt() = {}", ans);
}
```

しかしバイナリをみてみるとシンボルすら存在していない。

```console
$ cargo build -q
$ xtensa-esp32s3-elf-nm target/xtensa-esp32s3-espidf/debug/rust-esp32s3-example| grep sqrt
$
```

実体がないわけではないようだ。

```console
$ xtensa-esp32s3-elf-readelf -p .flash.rodata target/xtensa-esp32s3-espidf/debug/rust-esp32s3-example| grep sqrt
  [    68]  (5.0).sqrt() =
```

```console
$ xtensa-esp32s3-elf-objdump -d target/xtensa-esp32s3-espidf/debug/rust-esp32s3-example| grep -A 30 _ZN20rust_esp32s3_example4main17hf98358a820cefb84E
42004f00 <_ZN20rust_esp32s3_example4main17hf98358a820cefb84E>:
42004f00:	00a136        	entry	a1, 80
42004f03:	04c1a2        	addi	a10, a1, 4
42004f06:	ec4981        	l32r	a8, 4200002c <_stext+0xc> (420068c8 <_ZN11esp_idf_sys12link_patches17hffb97eb4add7e8c7E>)
42004f09:	0008e0        	callx8	a8
42004f0c:	ec4981        	l32r	a8, 42000030 <_stext+0x10> (420051cc <_ZN11esp_idf_svc3log9EspLogger18initialize_default17ha514c84e6cced950E>)
42004f0f:	0008e0        	callx8	a8
42004f12:	ec4841        	l32r	a4, 42000034 <_stext+0x14> (3fc93000 <_ZN3log20MAX_LOG_LEVEL_FILTER17h763732fb8bc56b2aE>)
42004f15:	0488      	l32i.n	a8, a4, 0
42004f17:	050c      	movi.n	a5, 0
42004f19:	160c      	movi.n	a6, 1
42004f1b:	ec4a71        	l32r	a7, 42000044 <_stext+0x24> (4200687c <_ZN3log13__private_api3log17h44c9357af96303c0E>)
42004f1e:	1c38b6        	bltui	a8, 3, 42004f3e <_ZN20rust_esp32s3_example4main17hf98358a820cefb84E+0x3e>
42004f21:	5159      	s32i.n	a5, a1, 20
42004f23:	2169      	s32i.n	a6, a1, 8
42004f25:	ec4481        	l32r	a8, 42000038 <_stext+0x18> (3c050148 <anon.803e1ef76bb95cd447cdd4924c3a9d53.0.llvm.7911434705372716145+0x28>)
42004f28:	1189      	s32i.n	a8, a1, 4
42004f2a:	4159      	s32i.n	a5, a1, 16
42004f2c:	ec4481        	l32r	a8, 4200003c <_stext+0x1c> (3c050138 <anon.803e1ef76bb95cd447cdd4924c3a9d53.0.llvm.7911434705372716145+0x18>)
42004f2f:	3189      	s32i.n	a8, a1, 12
42004f31:	04c1a2        	addi	a10, a1, 4
42004f34:	3b0c      	movi.n	a11, 3
42004f36:	ec42c1        	l32r	a12, 42000040 <_stext+0x20> (3c050170 <anon.803e1ef76bb95cd447cdd4924c3a9d53.0.llvm.7911434705372716145+0x50>)
42004f39:	9d0c      	movi.n	a13, 9
42004f3b:	0007e0        	callx8	a7
42004f3e:	ec4281        	l32r	a8, 42000048 <_stext+0x28> (400f1bbd <rom_rx_gain_force+0xeb791>)
42004f41:	0189      	s32i.n	a8, a1, 0
42004f43:	0488      	l32i.n	a8, a4, 0
42004f45:	290c      	movi.n	a9, 2
42004f47:	26b987        	bgeu	a9, a8, 42004f71 <_ZN20rust_esp32s3_example4main17hf98358a820cefb84E+0x71>
42004f4a:	5159      	s32i.n	a5, a1, 20
42004f4c:	2169      	s32i.n	a6, a1, 8
42004f4e:	ec3f81        	l32r	a8, 4200004c <_stext+0x2c> (3c050198 <anon.803e1ef76bb95cd447cdd4924c3a9d53.0.llvm.7911434705372716145+0x78>)
42004f51:	1189      	s32i.n	a8, a1, 4
42004f53:	4169      	s32i.n	a6, a1, 16
42004f55:	1cc182        	addi	a8, a1, 28
42004f58:	3189      	s32i.n	a8, a1, 12
42004f5a:	ec3d81        	l32r	a8, 42000050 <_stext+0x30> (42032cd8 <_ZN4core3fmt5float52_$LT$impl$u20$core..fmt..Display$u20$for$u20$f32$GT$3fmt17hc35c15e3727bc05dE>)
42004f5d:	8189      	s32i.n	a8, a1, 32
42004f5f:	00c182        	addi	a8, a1, 0
42004f62:	7189      	s32i.n	a8, a1, 28
42004f64:	04c1a2        	addi	a10, a1, 4
42004f67:	3b0c      	movi.n	a11, 3
42004f69:	ec35c1        	l32r	a12, 42000040 <_stext+0x20> (3c050170 <anon.803e1ef76bb95cd447cdd4924c3a9d53.0.llvm.7911434705372716145+0x50>)
42004f6c:	cd0c      	movi.n	a13, 12
42004f6e:	0007e0        	callx8	a7
42004f71:	f01d      	retw.n
```

## まとめ？

esp-rsエコシステム複雑過ぎてよくわからん

[espup]: https://github.com/esp-rs/espup
[rustbuild]: https://github.com/esp-rs/rust-build
[llvm_xtensa_release_17.0.1]: https://github.com/espressif/llvm-project/tree/xtensa_release_17.0.1
