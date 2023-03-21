---
title: "go_buildの話"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go]
published: false
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

(めちゃくちゃ余談ですがChatGPTにログからプライベートっぽい情報を消してと頼んだらこうなりました。ユーザー名など消えててそれっぽいなと思いました)

## ログを読み解く

ログを見ていくとちょくちょく`mkdir -p $WORK/b005`のようにディレクトリを作っています。
これはビルドプロセスの実行単位ごとに行われており、今回の場合b001 ~ b040まであります。本記事ではこの実行単位をステップと呼ぶことにします。

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

- go tool compilerを呼び出してソースコードをアセンブリ言語に変換するステップ
- go tool asmとgo tool packを呼び出してアーカイブファイルを作成するステップ
- go tool linkを呼び出して実行ファイルを作成するステップ（b001/exec）

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


## go tool compilerを呼び出してソースコードをアセンブリ言語に変換するステップ
## go tool asmとgo tool packを呼び出してアーカイブファイルを作成するステップ
## go tool linkを呼び出して実行ファイルを作成するステップ（b001/exec）

