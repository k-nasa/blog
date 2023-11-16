---
title: "RISC-V Overview for os dev"
emoji: "🐷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---


## アウトライン

- まえがき
    - 筆者について
        - OS開発をはじめた
        - RISC-V向けOSをRustで書いている
        - ページング、例外などCPU機能を活用しようとすると苦戦
        - RISC-Vでこれらをどうやるのかまとめておきたかった
        - また、RISC-Vの全体像も知っておきたい
    - OS開発初心者向けに

## memo


## 命令セット

命令セットは基本命令と拡張命令によって構成されています。

基本命令はIのsuffixがついており、それぞれ32, 64, 128bitが定義されています。

- RV32I
- RV64I
- RV128I

### 拡張命令

基本命令の拡張としてこれらが定義されています。

- M: 整数の掛け算割り算
- A: アトミック操作(不可分操作)
- F, D, Q: 単精度浮動小数点数(float), 倍精度浮動小数点(double), 4倍精度浮動小数点数(quad)
- G: 一般的な用途を想定したもの。IMAFDをあわせてG
- C: 16bit短縮命令

上記の他にもいくつか拡張命令が存在します。

## 汎用レジスタ

x0からx31までの32個の汎用レジスタがある。

| register | ABI | description |
| --- | --- | --- |
| x0 | zero | 常に0になってるレジスタ |
| x1 | ra | 戻り先のアドレス(Return Address) |
| x2 | sp | スタックポインタ|
| x3 | gp | グローバルポインタ|
| x4 | tp| スレッドポインタ |
| x5 | t0 | 一次保存用のレジスタ・または第二のリンクレジスタ |
| x6-7 | t1-2| 一次保存用 |
| x8 | s0/fp| Callee savedレジスタ・またはframe pointer |
| x9 | s1 |  Callee savedレジスタ |
| x10-11 | a0-1| 関数の引数・または返り値 |
| x12-17 | a2-7| 関数の引数 |
| x18-27 | s2-11| Callee savedレジスタ  |
| x28-21 | t3-6| 一次保存用 |


### リンクレジスタ

Return Addressを格納するレジスタをリンクレジスタを呼ぶようです。
リンクレジスタがx1(ra)とx5(t0)の２つ存在することが気になりましたがこの理由はよく分かりませんでした。

### Calle savedレジスタ

呼び出された関数が呼び出し元の関数レジスタ内容を維持する必要があるレジスタをCallee savedレジスタと呼びます。

具体例を上げます。
callerがs0に5を格納していた場合を考えます。このときcalleeがs0を利用し値を書き換えた場合には、関数終了時にs0 = 5を復元する必要があるということです。


## 権限モード

動作モードと呼んでいるのをよく見ますが権限と言う方が好みなので権限モードを呼びます。
権限モードは3つありそれぞれ

- U-mode: ユーザーモード
- S-mode: スーパバイザーモード。
- M-mode: マシンモード。何でも出来る


`ECALL`命令により一つ上の権限モードに遷移する.

## 例外・割り込み


## 参考

- [Linux on RISC-V](https://kernel-recipes.org/en/2022/talks/linux-on-risc-v/)
- [The RISC-V Instruction Set Manual Volume II: Privileged Architecture](https://drive.google.com/file/d/1EMip5dZlnypTk7pt4WWUKmtjUKTOkBqh/view)
- [xv6: a simple, Unix-like teaching operating system](https://pdos.csail.mit.edu/6.828/2022/xv6/book-riscv-rev3.pdf)
