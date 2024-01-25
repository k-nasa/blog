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
    - 「可能な限り意味のある結果」をどうやって組み立てているのか？
    - Prismの動かし方

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
Rubyでよくやってしまう構文エラーをいくつか挙げデフォルトパーサーとPrismとでエラーメッセージを見比べてみます。

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
Prismに関しては２つ目の構文エラーに関しても扱えているのでこの点は良いなって思いました。


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

これはデフォルトパーサーがプログラマーが意図してそうな変数名をサジェストしてくれてていい感じですね。


| Default parser | Prism |
| --- | --- |
| ![](/images/ruby_prism_begin/r_typo_variable.png)| ![](/images/ruby_prism_begin/p_typo_variable.png) |


続いてメソッド名のタイポもやってみます。

```ruby
def tes
end

test1
```

メソッドも同様デフォルトパーサーが良い感じですねー

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


### 所感

RubyのデフォルトパーサーとPrismのエラーメッセージを見比べてみての所感ですが

- Prismで複数箇所のエラーを扱ってくれるのは便利. デフォルトパーサーでは1つめのエラー時点で終了してしまう
- Prismはクラッシュする問題があって現時点では厳しい(リリースノートにちゃんとそう書いてありますがね、、)
    - リリース直後なので当たり前ですね。今後の改善に期待

現時点ではどちらのほうが開発体験が良さそうとも言えない状況だなーと感じました。
今後の改善が楽しみですね。

---

本編は以上でここからは余談的なところです。

- 「可能な限り意味のある結果」をどうやって組み立てているのか？
- Prismの動かし方

について扱いますー


## 「可能な限り意味のある結果」をどうやって組み立てているのか？

どうやって正しい構文を推論し意味のある結果を返しているのか気になったのでソースコードを読んで調べました。(それっぽいドキュメントがなかったため)

ここでは該当箇所を見つける過程と実装の解説を少し行います。


### エラートレラントのエントリーポイントを探す

最初にエラートレラントを行っている関数があると仮説を立ててコードを読んでいきました。
結果としてエラートレラント関数のようなものは存在しておらず、トークン列を辿っていき都度正しい構文で埋めていることが分かりました。

仮説は外れていましたが`parse_statements`を読めば仕組みを把握できそうなことが分かりました

`Prism.parse`メソッドを起点にした場合下記の流れで`parse_statements`までたどり着けます。

- `Prism.parse` (ここまでRuby)
- [parse関数](https://github.com/ruby/prism/blob/3f00d9f0743c948f2c1768dce4716ff499b927ce/ext/prism/extension.c#L632-L679) (こっからC)
- [parse_input関数](https://github.com/ruby/prism/blob/3f00d9f0743c948f2c1768dce4716ff499b927ce/ext/prism/extension.c#L605-L610)
- [pm_parse](https://github.com/ruby/prism/blob/3f00d9f0743c948f2c1768dce4716ff499b927ce/src/prism.c#L17732-L17735)
- [parse_program](https://github.com/ruby/prism/blob/3f00d9f0743c948f2c1768dce4716ff499b927ce/src/prism.c#L17490-L17515)
- [parse_statements](https://github.com/ruby/prism/blob/3f00d9f0743c948f2c1768dce4716ff499b927ce/src/prism.c#L11151-L11221)

## `defined?`キーワードのパースを読む

`parse_statements`のすべてを読むのは厳しいので`defined?`キーワードのみ見てみます。
`defined?`を選んだ理由は仕様が少なくパーサーの実装が簡素だったからです。興味があれば`def`や`do`などを読んでみると学びが多そうです。


コードは[こちら](https://github.com/ruby/prism/blob/3f00d9f0743c948f2c1768dce4716ff499b927ce/src/prism.c#L15388-L15420)です。

```c
case PM_TOKEN_KEYWORD_DEFINED: {
    // 略...

    // 括弧が省略されていない時
    // ex. defined?(expression)
    if (accept1(parser, PM_TOKEN_PARENTHESIS_LEFT)) {
        lparen = parser->previous;

        // 式のパース。今回は本題じゃない
        expression = parse_expression(parser, PM_BINDING_POWER_COMPOSITION, true, PM_ERR_DEFINED_EXPRESSION);

        // `parser->recovering`はエラーから復帰中のときにtrueになる
        // どのトークンで埋めるか自明じゃないときにcontextにそれを残しておいてどっかでリカバリーするっぽい？
        if (parser->recovering) {
            rparen = not_provided(parser);
        } else {
            // 指定したトークンであれば読み飛ばす。ここでは改行であれば読み飛ばす
            // 下記の書き方に対応するため
            // ex.
            // ```ruby
            // defined?(
            //  expression
            //            ^ ここの改行に対応
            // )
            // ```
            accept1(parser, PM_TOKEN_NEWLINE);

            // ********************
            // **** ここが本題 ****
            // ********************
            // 閉じ括弧括弧が存在するか確認し存在しなければ第三引数`PM_ERR_EXPECT_RPAREN`エラーを`parser`構造体のerrorsに追加している。
            // その後`PM_TOKEN_MISSING`で埋めている
            // 重要なのは次の二点だと解釈している
            // 1. エラーをparser->errorsに蓄積すること
            // 2. TOKEN_MISSINGで埋めてパース処理を続行すること
            expect1(parser, PM_TOKEN_PARENTHESIS_RIGHT, PM_ERR_EXPECT_RPAREN);
            rparen = parser->previous;
        }
    // 括弧の省略時
    // ex. defined? expression
    } else {
        // not_providedは括弧が省略されていることを示すトークンを返す関数
        lparen = not_provided(parser);
        rparen = not_provided(parser);

        // 式のパース。今回は本題じゃない
        expression = parse_expression(parser, PM_BINDING_POWER_DEFINED, false, PM_ERR_DEFINED_EXPRESSION);
    }

    return (pm_node_t *) pm_defined_node_create(
        parser,
        &lparen,
        expression,
        &rparen,
        &PM_LOCATION_TOKEN_VALUE(&keyword)
    );
}
```

基本的な方針としては必要なトークンがないときはエラーを配列に詰め、足りないトークンをPM_TOKEN_MISSINGで埋め可能な限り処理を続行することです。

僕の持っていた仮説は「構文情報というものがありそこから論理的に近しいトークンを推論する」だったのですが、理論ベースではなく直感的に正しいもので埋めておくってことをしていました。
「このケースはエラーですよね。あなたがやりたいことはこれですよね」って感じなんですね。

## Prismの動かし方

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

SEGV received in BUS handler
```

