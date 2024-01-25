---
title: "Ruby 3.3で導入されたPrismを使うとRubyの開発体験がどう変わるのか"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## タイトル候補

- Ruby3.3で導入されたPrismを使うとRubyの開発体験がどう変わるのか
- 新しいRubyパーサーPrismを使うと開発体験がどう変わるのか
- 募集中。。。


## この記事で扱っていること

- Ruby 3.3でPrismという新しいパーサーが追加されました
- 本記事ではPrismの「パース時に問題が発生した場合でも可能な限り意味のある結果を返却する」という機能に着目して下記について話します
    - Rubyを記述する際の開発者体験がどれくらい向上しているのか？
    - 「可能な限りいものある結果」をどうやって組み立てているのか？

## はじめに

こんにちはウォンテッドリーで推薦基盤の改善をやっている[nasa](https://twitter.com/nasa_desu)です。

去年もRubyの最新バージョンがサンタさんから届きました。https://www.ruby-lang.org/ja/news/2023/12/25/ruby-3-3-0-released/
Ruby 3.3ではPrismという新しいパーサーが実験的に導入されています。

Prismはデフォルトでは有効になっていませんが`ruby --parser=prism`とオプションを指定することで使える状態です。
デフォルトgemとしても導入されているので下記のようなRubyプログラムで試すことも出来ます。

```ruby
require "prism"

source = <<~RUBY
def foo
  puts "foo"
end
RUBY

result = Prism.parse(source)
```


本記事ではPrismの「パース時に問題が発生した場合でも可能な限り意味のある結果を返却する」という機能について僕の疑問をまとめます。

## Prismの特徴

新しいパーサーということでまずはPrismの概要を述べておきます。本題に早く入りたい人は適当に読み飛ばして下さいー

Prismは主にShopifyが開発している新しいRubyパーサーです。
現時点のRubyの構文は100%カバーしているのとShopifyなど複数の大規模アプリケーションで動作確認を行い、問題なく動作することが確認できているステータスのようです。

Prismは下記の3つが開発のモチベーションとなっています。

- エラートレラント
- ポータビリティ
- メンテナンス性

それぞれ少し詳しく書いていきますね。

### エラートレラント

エラートレラントはその名の通り、パーサーで何か問題が発生した場合でも可能な限り意味のある結果を返却することです。

パーサーの主な責務はASTを組み立てることですが、LSPのような実装途中のプログラムを扱うソフトウェアに関してはただASTを組み立てるだけでは不十分です。
ソースコードが絶え間なく変化する状況で開発者にとって有益な情報を返すことが望まれます。

### ポータビリティ

現状のRubyのパーサーはCRuby, JRuby, TruffleRubyといったRuby処理系、sorbet, steep, ruby-lspなどの開発ツールなど多くに実装されています。
Prismはポータビリティを念頭に実装されておりこのカオスを打破することを目指しているようです。

### メンテナビリティ

ポータビリティの結果、多くのコミュニティーで長く使われてることを想定しており結果としてメンテナンス性の重要度が上がっているって感じみたいです。

実際にドキュメントやテストの拡充には力を入れている印象です。

これらがPrismが開発されたモチベーションになります。
Prism開発者が公開しているこちらのスライドを見るとより多くを学べると思います！

https://speakerdeck.com/kddnewton/prism

## Rubyを記述する際の開発者体験がどれくらい向上しているのか？

本題です。ではPrismのエラートレラントによってRubyを使用した開発体験がどの程度向上するのか見ていきましょう。

Rubyでよくやってしまう構文エラーをいくつか上げデフォルトパーサーとPrismとで挙動の違いを見てみます。

### end忘れ

ネストが深くなったときによくやってしまいます。endが一つ多いのもやる。。

```ruby
def example_method
  puts "Hello"
```

これはPrismのほうが対応が必要な箇所が分かりやすいですね

| Default parser | Prism |
| --- | --- |
| ![](/images/ruby_prism_begin/r_endless.png)| ![](/images/ruby_prism_begin/p_endless.png) |

### endが多い

ということで先程出たendが一つ多い問題


```ruby
def example_method
  puts "Hello"
end
end
```

デフォルトパーサーのエラーメッセージ`unexpected end`は分かりやすいですがエラー箇所把握がちょっとムズい。
Prismのエラー箇所は的を射ているがエラーメッセージ`cannot parse the expression`は不正確ですね。endキーワードではなく式としてパースされているのでしょうか

| Default parser | Prism |
| --- | --- |
| ![](/images/ruby_prism_begin/r_endmore.png)| ![](/images/ruby_prism_begin/p_endmore.png) |



### 閉じ括弧忘れ

```ruby
def test(
  puts "hoge"
end

test(
```

どちらも`puts`を引数として解釈して`puts`の後に閉じ括弧が必要だと解釈しているようですね。
こちらもトントンですかね。
Prismに関しては２つ目の構文エラーに関しても扱えているのでこの点はめちゃ良いですねー


| Default parser | Prism |
| --- | --- |
| ![](/images/ruby_prism_begin/r_endmore.png)| ![](/images/ruby_prism_begin/p_endmore.png) |


### 区切り文字忘れ


```ruby
array = [1, 2, 3, 4 5]
```

Prismのエラーメッセージも指摘箇所も非常に分かりやすいですね

| Default parser | Prism |
| --- | --- |
| ![](/images/ruby_prism_begin/r_delimiter.png)| ![](/images/ruby_prism_begin/p_delimiter.png) |


### 変数名タイポ(存在しない変数名)

```ruby
a = 1
puts aa
```

おお！これはデフォルトパーサーが良い感じですね。


| Default parser | Prism |
| --- | --- |
| ![](/images/ruby_prism_begin/r_typo_variable.png)| ![](/images/ruby_prism_begin/p_typo_variable.png) |


続いてメソッド名のタイポもやってみます。

```ruby
def tes
end

test1
```

関数も同様デフォルトパーサーが良い感じですねー

| Default parser | Prism |
| --- | --- |
| ![](/images/ruby_prism_begin/r_typo_method.png)| ![](/images/ruby_prism_begin/p_typo_method.png) |


### メソッド呼び出しのドットだけ書いた

あまり無いミスですが実装途中によく発生すると思うので試してみます。

```ruby
test.
```


おっと。Prismはクラッシュしてしまいましたね。。。
PRチャンスかもしれません。

| Default parser | Prism |
| --- | --- |
| ![](/images/ruby_prism_begin/r_dot.png)| ![](/images/ruby_prism_begin/p_dot.png) |

### インデントを考慮してくれるか

このようなコードではインデントを考慮するとifに対するendが無いと言えます。この辺考慮してくれるのでしょうか

```ruby
def test
  if true
    puts "true"
end
```

まあ予想はしていましたが厳しいようですね。。。


| Default parser | Prism |
| --- | --- |
| ![](/images/ruby_prism_begin/r_indent.png)| ![](/images/ruby_prism_begin/p_indent.png) |


### その他

ここで扱ったのはほんの一部になります。

その他に対応しているエラーについてはPrismのテストコードを眺めてみて下さい！

https://github.com/ruby/prism/blob/3f00d9f0743c948f2c1768dce4716ff499b927ce/test/prism/errors_test.rb


---


### Prismの動かし方

rubyインタプリタ経由で動かすのに一工夫必要だったのでそれを紹介します。(本題じゃなくてごめんなさい)

`ruby --parser=prism`とオプションを指定することで利用はできるんですが、構文エラーが発生した際にはクラッシュしてしまう状態です。。。(このバグは3.3.1で修正予定みたい)

なのでmasterのrubyを利用する必要があります。
実際に動かしてみたい人はこちらのドキュメントを頼りに最新のRubyをビルドしてみて下さい！

https://docs.ruby-lang.org/en/master/contributing/building_ruby_md.html


それから実はもう一つハマりポイントがあります。ファイルの末尾の空行の有無によって挙動が変わるため注意が必要です。
空行が存在する場合はきちんと動いてくれるんですが、空行があるとハングしてしまいました。これもおそらく3.3.1で修正されると思います。


### Prismが動作するプログラム(空行あり)

```ruby
:) % bat main.rb
───────┬──────────────────────────────
       │ File: main.rb
───────┼──────────────────────────────
   1   │ class Foo
   2   │   def initialize(
   3   │     true
   4   │   end
   5   │
───────┴──────────────────────────────

:) % ruby --parser=prism main.rb
main.rb: syntax errors found (SyntaxError)
  1 | class Foo
> 2 |   def initialize(
    |                  ^ expected a `)` to close the parameters
  3 |     true
  4 |   end
> 5 |
    | ^ expected an `end` to close the `class` statement
    | ^ cannot parse the expression
```

### Prismがクラッシュするプログラム(空行なし)

```
:) % bat main.rb
───────┬──────────────────────────────
       │ File: main.rb
───────┼──────────────────────────────
   1   │ class Foo
   2   │   def initialize(
   3   │     true
   4   │   end
───────┴──────────────────────────────

:) % /Users/wantedly564/lab/oss/ruby/build/ruby --parser=prism main.rb
SEGV received in BUS handler
zsh: abort      /Users/wantedly564/lab/oss/ruby/build/ruby --parser=prism main.rb
```

