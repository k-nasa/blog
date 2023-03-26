---
title: "go buildは何をやっている？"
emoji: "💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go]
published: true
---

goコンパイラはソースコードから実行ファイルを作ってくれていますが裏側で何をしているのか謎です。
`go build`コマンドの裏側をログ出力を元に理解していきます。

## ログを出力する

利用したコード下記のとおりです。
世界に挨拶するプログラムをコンパイルし、そのコンパイルログを見ていきます。

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello World")
}
```


実行コマンドは下記のとおりです。`-x`オプションを付けることで大量のログが出力されます。

```sh
go build -x -p 1 main.go

# -x: 実行しているコマンドを出力する
# -p 1: シングルスレッドで実行する（ログが読み取りやすくなる）
```


こんなログが出力されます。
ここではログが長いため省略していますが、興味があれば手元で実行してみてください。このログをもとに、ビルドプロセスを理解していきます。


```
WORK=/var/folders/xx/xxxxxxxxxxxxx/T/go-buildxxxxxxxx
mkdir -p $WORK/b005/
cat >/var/folders/xx/xxxxxxxxxxxxx/T/go-buildxxxxxxxx/b005/importcfg << 'EOF' # internal
# import config
EOF
cd /Users/xxxxx/lab/sandbox/playground_go
/usr/local/go/pkg/tool/xxxxxxxx/compile -o $WORK/b005/_pkg_.a -trimpath "$WORK/b005=>" -p internal/goarch -std -+ -complete -buildid M8znE3RFNIIsfAk_eE8O/M8znE3RFNIIsfAk_eE8O -goversion go1.20.1 -c=8 -nolocalimports -importcfg $WORK/b005/importcfg -pack /usr/local/go/src/internal/goarch/goarch.go /usr/local/go/src/internal/goarch/goarch_amd64.go /usr/local/go/src/internal/goarch/zgoarch_amd64.go
/usr/local/go/pkg/tool/xxxxxxxx/buildid -w $WORK/b005/_pkg_.a # internal
cp $WORK/b005/_pkg_.a /Users/xxxxx/Library/Caches/go-build/8c/8cf7915e43e6ec3a1ceac20e15009d87c0d26d4aa40409e5dc8b2e935d6d1014-d # internal
mkdir -p $WORK/b006/
cat >/var/folders/xx/xxxxxxxxxxxxx/T/go-buildxxxxxxxx/b006/importcfg << 'EOF' # internal
# import config
EOF
/usr/local/go/pkg/tool/xxxxxxxx/compile -o $WORK/b006/_pkg_.a -trimpath "$WORK/b006=>" -p internal/unsafeheader -std -complete -buildid sD4mccwa9DUJsmqBZGxG/sD4mccwa9DUJsmqBZGxG -goversion go1.20.1 -c=8 -nolocalimports -importcfg $WORK/b006/importcfg -pack /usr/local/go/src/internal/unsafeheader/unsafeheader.go
/usr/local/go/pkg/tool/xxxxxxxx/buildid -w $WORK/b006/_pkg_.a # internal
...
...
...
```

## ログを読み解く

ログを見ていくとちょくちょく`mkdir -p $WORK/b005`と作業ディレクトリを作っています。
これはビルドプロセスの実行単位ごとに行われており、今回の場合b001 ~ b040までありました。本記事ではこの実行単位をステップと呼ぶことにします。

goの1パッケージごとに1ステップ存在すると想像しています。また並列でビルドする場合には各ステップが並列で動作します。

ステップの生成物は他のステップで使い回され最終的には`b001/exec`というステップで実行ファイルが作られます。
その様子を図にしてみましたが理解できたもんじゃないですね。ステップに依存関係があり最終的に`b001`に集約されるんだーくらいの気持ちで眺めて下さい

![ステップの依存関係](/images/build-prosess-detail.png)


各ステップでの処理は次の9つの中からいくつか行われます。
1,2, 7,8はすべてのステップで共通で行われますがその他は状況に応じて実行されます。

1. 作業ディレクトリ作成
1. importcfgの作成
1. ヘッダーファイルの作成
1. go tool compileを呼び出しアセンブリファイルを作成
1. go tool asmを呼び出しオブジェクトファイルを作成
1. go tool packを呼び出しオブジェクトファイルをアーカイブファイルに追加
1. go tool buildidを呼び出してキャッシュのキー情報を書き出す
1. キャッシュをcache用ディレクトリへの待避
1. go tool linkを呼び出して実行ファイルを作成



これらの処理がどのステップで行われているか見ていくと下記の表のようになります。

表を見るとビルドプロセスは次の3つの処理に分類できそうです。

- go tool compilerを呼び出してソースコードをオブジェクトファイルに変換するステップ
- go tool asmとgo tool packを呼び出してアーカイブファイルを作成するステップ
- go tool linkを呼び出して実行ファイルを作成するステップ（これは最後のステップb001/execのみ）

| stage | compile | link | header file | asm | pack | depend steps |
| --- |  --- | --- | --- | --- | --- | --- |
| b005/ | o | | | |  | |
| b006/ | o | | | |  | |
| b009/ | o | | o | o | o | | b005/ |
| b011/ | o | | o | o | o |  | |
| b010/ | o | | o | o | o | b011/ |
| b031/ | o | | | | | b008/, b003/, b020/, b027/, b025/, b029/, b030/, b034/, b033/, b036/, b032/, b037/, b038/, b039/, b040/ |
| 略  | | | | | | |
| b002/ | o | | | | | b003/, b021/, b024/, b023/, b025/, b019/, b029/, b018/, b030/, b031/ |
| b001/ | o | | | | | b005/, b006/, b009/, b011/, b010/, b012/, b013/, b014/, b015/, b016/, b017/, b008/, b004/, b003/, b020/, b022/, b021/, b024/, b023/, b026/, b027/, b025/, b028/, b019/, b029/, b018/, b030/, b035/, b034/, b033/, b036/, b032/, b037/, b038/, b039/, b041/, b040/, b031/, b002/ |
| b001/exe/ | | o | | | | b001/ |


ここまででビルドプロセスの全体感を把握できたと思います。
次に先程あげた3つ分類と全ステップの共通処理について詳細に見ていきます。

## 全ステップ共通処理

1. 作業ディレクトリ作成
1. importcfgの作成
1. go tool buildidを呼び出してキャッシュのキー情報を書き出す
1. キャッシュをcache用ディレクトリへの待避

作業ディレクトリ作成はその名の通りです。


> 1. importcfgの作成

次にimportcfgの作成ですが、importcfgはインポートされるパッケージの情報と、そのパッケージがどこに存在するかを含んでいます。
このファイルはパッケージの依存関係を正しく構築するために使用されます。

importcfgの中身は下記のようになっているようです。

```sh
$ go build -x -p 1 main.go
...
...
cat >/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b008/importcfg << 'EOF' # internal
# import config
packagefile internal/abi=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b009/_pkg_.a
packagefile internal/bytealg=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b010/_pkg_.a
packagefile internal/coverage/rtcov=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b012/_pkg_.a
packagefile internal/cpu=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b011/_pkg_.a
packagefile internal/goarch=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b005/_pkg_.a
packagefile internal/goexperiment=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b013/_pkg_.a
packagefile internal/goos=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b014/_pkg_.a
packagefile runtime/internal/atomic=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b015/_pkg_.a
packagefile runtime/internal/math=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b016/_pkg_.a
packagefile runtime/internal/sys=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b017/_pkg_.a
EOF
```

> 1. go tool buildidを呼び出してキャッシュのキー情報を書き出す
> 1. キャッシュをcache用ディレクトリへの待避

最後にビルドキャッシュ周りの処理ですが、これは下記のようなコマンドが実行されています。


```sh
/usr/local/go/pkg/tool/darwin_amd64/buildid -w $WORK/b001/_pkg_.a # internal
cp $WORK/b001/_pkg_.a /Users/xxx/Library/Caches/go-build/93/93acd2ffef4b3e3232b1fb6aa9c33c0a984f684d24394bafa05bbb6c6d5bfb1a-d # internal
```

雰囲気、ソースコードに応じてIDの発行・書き込みがなされ、それをCachesディレクトリに保存しておくことで次のコンパイル時間を短縮しているようですね。


詳しい話はbuildidコマンドのドキュメントコメントにあります。今回は省略します

https://github.com/golang/go/blob/master/src/cmd/go/internal/work/buildid.go


## go tool compilerを呼び出してソースコードをオブジェクトファイルに変換するステップ


下記は`fmt`パッケージをコンパイルしているログです。

オプションがいっぱいあるのでややこしいですが`fmt`パッケージに含まれるソースコード`/usr/local/go/src/fmt/doc.go /usr/local/go/src/fmt/errors.go /usr/local/go/src/fmt/format.go /usr/local/go/src/fmt/print.go /usr/local/go/src/fmt/scan.go`をコンパイルして`b002/_pkg.a_`に出力しているだけです。

```sh
mkdir -p $WORK/b002/

# (略) importcfg作成

/usr/local/go/pkg/tool/darwin_amd64/compile -o $WORK/b002/_pkg_.a -trimpath "$WORK/b002=>" -p fmt -std -complete -buildid ANLxqduoBT1SlW2syPt0/ANLxqduoBT1SlW2syPt0 -goversion go1.20.1 -c=8 -nolocalimports -importcfg $WORK/b002/importcfg -pack /usr/local/go/src/fmt/doc.go /usr/local/go/src/fmt/errors.go /usr/local/go/src/fmt/format.go /usr/local/go/src/fmt/print.go /usr/local/go/src/fmt/scan.go

# (略) go tool buildidを呼び出してキャッシュのキー情報を書き出す
# (略) キャッシュをcache用ディレクトリへの待避
```

## go tool asmとgo tool packを呼び出してアーカイブファイルを作成するステップ

これはアセンブラ実装を含むパッケージのためのステップです。
例えば、internal/abiやinternal/cpuは一部関数がアセンブラで実装されています。([internal/cpuのアセンブラ実装](https://cs.opensource.google/go/go/+/master:src/internal/cpu/cpu_x86.s))


```
mkdir -p $WORK/b011/

cat >/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b011/go_asm.h << 'EOF' # internal
EOF

cd /usr/local/go/src/internal/cpu
/usr/local/go/pkg/tool/darwin_amd64/asm -p internal/cpu -trimpath "$WORK/b011=>" -I $WORK/b011/ -I /usr/local/go/pkg/include -D GOOS_darwin -D GOARCH_amd64 -D GOAMD64_v1 -gensymabis -o $WORK/b011/symabis ./cpu.s ./cpu_x86.s

# (略) importcfg作成

cd /Users/xxx/lab/sandbox/playground_go

/usr/local/go/pkg/tool/darwin_amd64/compile -o $WORK/b011/_pkg_.a -trimpath "$WORK/b011=>" -p internal/cpu -std -+ -buildid OsYLB-sNg0tBjaiHx8em/OsYLB-sNg0tBjaiHx8em -goversion go1.20.1 -symabis $WORK/b011/symabis -c=8 -nolocalimports -importcfg $WORK/b011/importcfg -pack -asmhdr $WORK/b011/go_asm.h /usr/local/go/src/internal/cpu/cpu.go /usr/local/go/src/internal/cpu/cpu_x86.go

cd /usr/local/go/src/internal/cpu

/usr/local/go/pkg/tool/darwin_amd64/asm -p internal/cpu -trimpath "$WORK/b011=>" -I $WORK/b011/ -I /usr/local/go/pkg/include -D GOOS_darwin -D GOARCH_amd64 -D GOAMD64_v1 -o $WORK/b011/cpu.o ./cpu.s
/usr/local/go/pkg/tool/darwin_amd64/asm -p internal/cpu -trimpath "$WORK/b011=>" -I $WORK/b011/ -I /usr/local/go/pkg/include -D GOOS_darwin -D GOARCH_amd64 -D GOAMD64_v1 -o $WORK/b011/cpu_x86.o ./cpu_x86.s

/usr/local/go/pkg/tool/darwin_amd64/pack r $WORK/b011/_pkg_.a $WORK/b011/cpu.o $WORK/b011/cpu_x86.o # internal

# (略) go tool buildidを呼び出してキャッシュのキー情報を書き出す
# (略) キャッシュをcache用ディレクトリへの待避
```

やっていることは下記のとおりです。

- ヘッダファイル作成
- go tool asmを実行してsymabis ファイルを作成
- compileをを実行(このときヘッダファイルとsymabisを仕様)
- go tool asmを実行してアセンブリファイルをオブジェクトファイルにする(ここではcpu.sとcpu_x86.s)
- packを実行してオブジェクトファイルをアーカイブファイルに追加する

僕自身このステップはあまり理解しておらず、ヘッダファイルが必要な理由、symabisとは？などの疑問があります。。。

低レベルな関数だと複雑なんだなーと思っておくことにします。。


## go tool linkを呼び出して実行ファイルを作成するステップ（b001/exec）

これは最終ステップで実行ファイルを作るステップです。
これまで作ってきたアーカイブファイルがここにて集結します。

```
cat >/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b001/importcfg.link << 'EOF' # internal
packagefile command-line-arguments=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b001/_pkg_.a
packagefile fmt=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b002/_pkg_.a
packagefile runtime=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b008/_pkg_.a
packagefile errors=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b003/_pkg_.a
packagefile internal/fmtsort=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b018/_pkg_.a
packagefile io=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b030/_pkg_.a
packagefile math=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b021/_pkg_.a
packagefile os=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b031/_pkg_.a
packagefile reflect=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b019/_pkg_.a
packagefile sort=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b029/_pkg_.a
packagefile strconv=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b023/_pkg_.a
packagefile sync=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b025/_pkg_.a
packagefile unicode/utf8=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b024/_pkg_.a
packagefile internal/abi=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b009/_pkg_.a
packagefile internal/bytealg=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b010/_pkg_.a
packagefile internal/coverage/rtcov=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b012/_pkg_.a
packagefile internal/cpu=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b011/_pkg_.a
packagefile internal/goarch=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b005/_pkg_.a
packagefile internal/goexperiment=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b013/_pkg_.a
packagefile internal/goos=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b014/_pkg_.a
packagefile runtime/internal/atomic=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b015/_pkg_.a
packagefile runtime/internal/math=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b016/_pkg_.a
packagefile runtime/internal/sys=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b017/_pkg_.a
packagefile internal/reflectlite=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b004/_pkg_.a
packagefile math/bits=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b022/_pkg_.a
packagefile internal/itoa=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b020/_pkg_.a
packagefile internal/poll=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b032/_pkg_.a
packagefile internal/safefilepath=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b037/_pkg_.a
packagefile internal/syscall/execenv=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b038/_pkg_.a
packagefile internal/syscall/unix=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b033/_pkg_.a
packagefile internal/testlog=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b039/_pkg_.a
packagefile io/fs=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b040/_pkg_.a
packagefile sync/atomic=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b027/_pkg_.a
packagefile syscall=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b034/_pkg_.a
packagefile time=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b036/_pkg_.a
packagefile internal/unsafeheader=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b006/_pkg_.a
packagefile unicode=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b028/_pkg_.a
packagefile internal/race=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b026/_pkg_.a
packagefile internal/oserror=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b035/_pkg_.a
packagefile path=/var/folders/5r/00bxy2351234s9y9w1jpvxtm0000gn/T/go-build3110046836/b041/_pkg_.a
modinfo "0w\xaf\f\x92t\b\x02A\xe1\xc1\a\xe6\xd6\x18\xe6path\tcommand-line-arguments\nbuild\t-buildmode=exe\nbuild\t-compiler=gc\nbuild\tCGO_ENABLED=1\nbuild\tCGO_CFLAGS=\nbuild\tCGO_CPPFLAGS=\nbuild\tCGO_CXXFLAGS=\nbuild\tCGO_LDFLAGS=\nbuild\tGOARCH=amd64\nbuild\tGOOS=darwin\nbuild\tGOAMD64=v1\n\xf92C1\x86\x18 r\x00\x82B\x10A\x16\xd8\xf2"
EOF

mkdir -p $WORK/b001/exe/
cd .
/usr/local/go/pkg/tool/darwin_amd64/link -o $WORK/b001/exe/a.out -importcfg $WORK/b001/importcfg.link -buildmode=exe -buildid=r7bp_L-cPEhsa6CohccH/jzN2BgIeBal3b-ZH8O1I/41RAmNEaSe7ZXYUlhDyB/r7bp_L-cPEhsa6CohccH -extld=clang $WORK/b001/_pkg_.a

# (略) go tool buildidを呼び出してキャッシュのキー情報を書き出す
# (略) キャッシュをcache用ディレクトリへの待避
```

重要なのはこの部分で、`go tool link`に`main.go`が依存しているパッケージのアーカイブファイルを渡して実行ファイルを作らせています。

```
/usr/local/go/pkg/tool/darwin_amd64/link -o $WORK/b001/exe/a.out -importcfg $WORK/b001/importcfg.link -buildmode=exe -buildid=r7bp_L-cPEhsa6CohccH/jzN2BgIeBal3b-ZH8O1I/41RAmNEaSe7ZXYUlhDyB/r7bp_L-cPEhsa6CohccH -extld=clang $WORK/b001/_pkg_.a
```

## 最後に

ログから得られる情報を集めただけなので解像度は低いですが`go build`の挙動を少しは理解できたのではないでしょうか？

次回があれば`go tool compile`と`go tool link`が何をやっているかを見ていき実際にソースコードが実行ファイルになる過程を見ていこうと思います。
もしくは今回何も触れなかった、おジェクトファイル、アーカイブファイル話かもしれません。
