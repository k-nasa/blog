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

## Prismの動かし方

rubyインタプリタ経由で動かすのに一工夫必要だったのでそれを紹介します。(本題じゃなくてごめんなさい)

`ruby --parser=prism`とオプションを指定することで利用はできるんですが、構文エラーが発生した際にはクラッシュしてしまう状態です。。。(このバグは3.3.1で修正予定みたい)

なのでmasterのrubyを利用する必要があります。
実際に動かしてみたい人はこちらのドキュメントを頼りに最新のRubyをビルドしてみて下さい！

https://docs.ruby-lang.org/en/master/contributing/building_ruby_md.html


それから実はもう一つハマりポイントがあります。ファイルの末尾の空行の有無によって挙動が変わるため注意が必要です。
空行が存在する場合はきちんと動いてくれるんですが、空行があるとハングしてしまいました。これもおそらく3.3.1で修正されると思います。


### 動くコード

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

### 動かないコード

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


| Default parser | Prism |
| --- | --- |
| ![](/images/ruby_prism_begin/r_endmore.png)| ![](/images/ruby_prism_begin/p_endmore.png) |

デフォルトパーサーのエラーメッセージ`unexpected end`は分かりやすいですがエラー箇所把握がちょっとムズい。
Prismのエラー箇所は的を射ているが`cannot parse the expression`が分かりづらい

どちらかが確実に良いとはいえなさそう？まあトントンと行ったところでしょうか。


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


### 区切り文字忘れ


```ruby
array = [1, 2, 3, 4 5]
```

これはPrismが良いですねー。メッセージも指摘箇所も分かりやすいですね


### 変数名タイポ(存在しない変数名)


```ruby
a = 1
puts aa
```

おお！これはデフォルトパーサーが良い感じですね。


```ruby
def test
end

test1
```

関数も同様デフォルトパーサーのが良い感じですねー


### メソッド呼び出しのドットだけ書いた

あまり無いミスですが実装途中によく発生すると思うので試してみます。

```ruby
test.
```


おっと。Prismはクラッシュしてしまいましたね。。。
PRチャンスかもしれません。

### インデントを考慮してくれるか

このようなコードではインデントを考慮するとifに対するendが無いと言えます。この辺考慮してくれるのでしょうか


```ruby
def test
  if true
    puts "true"
end
```

まあ予想はしていましたが厳しいようですね。。。


### その他

ここで扱ったのはほんの一部だと思います。

その他に対応しているエラーについてはPrismのテストコードを眺めてみて下さい！

https://github.com/ruby/prism/blob/3f00d9f0743c948f2c1768dce4716ff499b927ce/test/prism/errors_test.rb

---



- タイトル: Ruby 3.3で導入されたPrismを使うとRubyを使った開発体験がどう変わるのか
- Prismとは
    - Ruby 3.3から導入されたパーサー
    - 前身はYARP
    - Prismは下記を目的として開発されている
        - エラートレラント
            - パース時に問題が発生した場合でも可能な限り意味のある結果を返却すること
        - ポータビリティ
            - 複数のRuby処理系、型チェッカー、lspなどそれぞれにパーサーが存在している状態
            - これらを1つのパーサーで置き換えられるようにしたい
        - メンテナンス性
            - ポータビリティの結果、多くのコミュニティーで長く使われてる
            - 結果としてメンテナンス性の重要度が上がっている
- 本記事ではエラートレラントについて扱う
    - Rubyユーザーの開発体験に最も影響を与えるのはエラートレラントだと思われる
        - パフォーマンスの影響も0ではないが僕自身課題に感じたことはないため省略
    - エラートレラントとは
- 動かし方
    - `--parser=prism`オプションを付与することで試すことが出来る
    - そのはずだったのだが、最新のRuby 3.3.0では構文エラーがあった際にクラッシュする
    - Rubyのmasterブランチで動作させる必要がある(minirubyでも可能)
- Rubyでよくある構文エラーに対してどの程度対応できるのか
    - [x] endを漏らした
    - [x] 閉じカッコ
    - [x] 配列、ハッシュの閉じカッコ
    - [x] 配列、ハッシュの区切り文字
    - [x] 存在しない変数、メソッド
    - [x] メソッド呼び出しのドットだけ書いた
    - [ ] インデントを見ているのか
    - [ ] 複数エラー
- これらのエラーメッセージがどのように作られているのか


## end忘れ

<table>
<tr>
<td> default parser </td> <td> Prism </td>
</tr>
<tr>
<td>

```sh
Unmatched keyword, missing `end' ?
> 1  def example_method

test.rb:5: syntax error, unexpected end-of-input, expecting `end' or dummy end (SyntaxError)
```

</td>
<td>

```sh
test.rb: syntax errors found (SyntaxError)
  3 |     puts "Hello"
  4 |   end
> 5 |
    | ^ expected an `end` to close the `def` statement
    | ^ cannot parse the expression
```

</td>
</tr>
</table>


## endが多い

<table>
<tr>
<td> default parser </td> <td> Prism </td>
</tr>
<tr>
<td>

```sh
Unmatched `end', missing keyword (`do', `def`, `if`, etc.) ?
> 1  def example_method
> 3  end
> 4  end

test.rb:4: syntax error, unexpected `end' (SyntaxError)
```

</td>
<td>

```sh
test.rb: syntax errors found (SyntaxError)
  2 |   puts "Hello"
  3 | end
> 4 | end
    | ^ cannot parse the expression
```

</td>
</tr>
</table>


## 閉じ括弧忘れ

<table>
<tr>
<td> default parser </td> <td> Prism </td>
</tr>
<tr>
<td>

```sh
Unmatched `(', missing `)' ?
> 1  def test(
> 3  end
> 5  test(

test.rb:2: syntax error, unexpected string literal, expecting ')' (SyntaxError)
  puts "test"
       ^
```

</td>
<td>

```sh
test.rb: syntax errors found (SyntaxError)
  1 | def test(
> 2 |   puts "test"
    |       ^ expected a `)` to close the parameters
  3 | end
  4 |
> 5 | test(
    |      ^ expected a `)` to close the arguments
  6 |
```

</td>
</tr>
</table>

## 区切り文字

<table>
<tr>
<td> default parser </td> <td> Prism </td>
</tr>
<tr>
<td>

```sh
expected a `,` separator for the array elements
> 1  array = [1, 2, 3, 4 5]

test.rb:1: syntax error, unexpected integer literal, expecting ']' (SyntaxError)
array = [1, 2, 3, 4 5]
```

</td>
<td>

```sh
test.rb: syntax errors found (SyntaxError)
> 1 | array = [1, 2, 3, 4 5]
    |                    ^ expected a `,` separator for the array elements
```

</td>
</tr>
</table>


## 存在しない変数


<table>
<tr>
<td> default parser </td> <td> Prism </td>
</tr>
<tr>
<td>

```sh
test.rb:2:in `<main>': undefined local variable or method `aa' for main (NameError)

puts aa
     ^^
Did you mean?  a
```

</td>
<td>

```sh
/Users/wantedly564/lab/sandbox/ruby_sandobx/test.rb:1:in `<compiled>': undefined local variable or method `aa' for main (NameError)
```

</td>
</tr>
</table>

## 存在しないメソッド

<table>
<tr>
<td> default parser </td> <td> Prism </td>
</tr>
<tr>
<td>

```sh
test.rb:4:in `<main>': undefined local variable or method `test1' for main (NameError)

test1
^^^^^
Did you mean?  test
```

</td>
<td>

```sh
/Users/wantedly564/lab/sandbox/ruby_sandobx/test.rb:3:in `<compiled>': undefined local variable or method `test1' for main (NameError)
```

</td>
</tr>
</table>



## 動作するコード例


<table>
<tr>
<td> Status </td> <td> Response </td>
</tr>
<tr>
<td>

```sh
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

</td>
<td>

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

</td>
</tr>
</table>

### メソッド呼び出しのドットだけ書いた


<table>
<tr>
<td> default parser </td> <td> Prism </td>
</tr>
<tr>
<td>

```sh
expected a method name
> 1  test.
> 2
```

</td>
<td>

```sh
ruby 3.4.0dev (2024-01-19T11:02:59Z master 7b0f6d6d94) +PRISM [arm64-darwin22]

-- Crash Report log information --------------------------------------------
   See Crash Report log file in one of the following locations:
     * ~/Library/Logs/DiagnosticReports
     * /Library/Logs/DiagnosticReports
   for more details.
Don't forget to include the above Crash Report log file in bug reports.
```

</td>
</tr>
</table>

### インデント


<table>
<tr>
<td> default parser </td> <td> Prism </td>
</tr>
<tr>
<td>

```sh
Unmatched keyword, missing `end' ?
> 1  def test
> 2    if true
> 4  end

test.rb:5: syntax error, unexpected end-of-input, expecting `end' or dummy end (SyntaxError)
```

</td>
<td>

```sh
test.rb: syntax errors found (SyntaxError)
  3 |     puts "true"
  4 | end
> 5 |
    | ^ expected an `end` to close the `def` statement
    | ^ cannot parse the expression
```

</td>
</tr>
</table>

---



<table>
<tr>
<td> default parser </td> <td> Prism </td>
</tr>
<tr>
<td>

```sh
```

</td>
<td>

```sh
```

</td>
</tr>
</table>
