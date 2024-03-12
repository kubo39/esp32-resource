# Rustはesp32s3でちゃんとFPUを使ってくれるのか？

Rustはesp32s3でちゃんとFloating Point Coprocessorを扱ってくれるのか？(正確にいうと、デフォルトで除算やsqrtなどの実装が最適化された実装になってくれるのか)というのが気になる。

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

こちらも手元の環境と合致していることが確認できる。

```console
$ xtensa-esp32s3-elf-gcc --version| head -1
xtensa-esp-elf-gcc (crosstool-NG esp-13.2.0_20230928) 13.2.0
```

これでビルド対象の[GCCのタグ](https://github.com/espressif/gcc/tree/esp-13.2.0_20230928)がわかった。なぜリンカとしてしか使わないのにレポジトリまで見に行く必要があるか？というのは後々触れていく。

## espressif/LLVM

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

[libgccのsqrtfルーチン](https://github.com/espressif/gcc/blob/esp-13.2.0_20230928/libgcc/config/xtensa/ieee754-sf.S#L1827-L1874)を有効にするマクロ定義はcrosstool-NGが参照している[xtensa-overlaysレポジトリの設定](https://github.com/espressif/xtensa-overlays/blob/dd1cf19f6eb327a9db51043439974a6de13f5c7f/xtensa_esp32s3/gcc/include/xtensa-config.h#L101)で定義されている。

## まとめ

Rust with esp32s3はデフォルトでlibgccが持っている、Floating Point Coprocessorに最適化された実装を利用しているので心配せずに使って問題ない。

-------------------------

[espup]: https://github.com/esp-rs/espup
[rustbuild]: https://github.com/esp-rs/rust-build
[llvm_xtensa_release_17.0.1]: https://github.com/espressif/llvm-project/tree/xtensa_release_17.0.1
