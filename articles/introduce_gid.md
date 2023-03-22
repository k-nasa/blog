---
title: "GitHubのTrackedIssueの関係を可視化するツールを作った"
emoji: "🐦"
type: "idea"
topics: ["cli", "githubactions", "github"]
published: true
---

こんにちは[nasaちゃん](https://twitter.com/nasa_desu)です。

今日は最近GitHubのTracked Issueの関係を可視化するツールを作ったので紹介をしようと思います。このツールは今の所技術的に面白いことはしていないので技術話は省略します。

https://github.com/k-nasa/gid

## 何が出来るか

次の３つが主な機能です。

- TrackedIssueの関係図がissue上にコメントされます
- open, closeやサブイシューの追加などの変更があった際に図が自動で更新されます
- クリックすることでそのissueに遷移します

プロジェクトの進行状況や全体像がいつでも把握できていいね！！というツールになっています。

![](https://user-images.githubusercontent.com/23740172/162580458-c81677c0-f171-4eda-8e8b-c9b9bff38691.png)


ではTrackedIssueとは何でしょうか？GitHubの新し目の機能なので馴染みのない人もいるかと思います。

GitHubには、issueのbodyにチェックリストとして追加しているissueのopen, closeによってチェックリストの状態が変わる機能があります。またbodyにあるタスクリストをtracked issueとして変換することも出来ます。
(詳しくはGitHubの[ドキュメント](https://docs.github.com/ja/enterprise-cloud@latest/issues/tracking-your-work-with-issues/about-task-lists)を見てみてください！)


![](https://storage.googleapis.com/zenn-user-upload/aae93407fb4b-20220416.png)

## なぜ作ったか

僕はこの機能を使って1つの大きな問題に紐づくサブイシューを管理していました。

大きめのプロジェクトを進める際になるべく小さい問題(サブイシュー)に分割して進めていたのですが、issueの数が増えたとき、ネストが深くなったときなど複雑になってくると色々と不安が湧いてきます

- プロジェクトが今どうなっているのか？(進めている僕もふわっとしてくるしプロジェクトマネージャーも分からなくなる)
- 問題の切り方や進み方が正しそうか?
- 親イシューの課題に沿った解決になっているのか？(だんだん本来やりたかったことから逸れる危険性)

これらの不安を払拭するためにdraw.ioやmiroなどの作図ツールを使って図を更新しながらプロジェクトを進めることが多かったです。

結果、良い進め方が出来たと思っていますが作図が面倒くさいという思いもありました。

もともとGitHubのTrackedIssueを使ってissueの関係を管理していたのでこれをもっと利用できないかと考えツールを作りました。

## 使い方

次のようなGitHub Actions ワークフローを追加することで使えるようになります。

特定のラベル(下の例だとroot)が付いているissueをルートイシューとして解釈します。そしてそのルートイシューにコメントが追加されます。

issueのopen, close,ラベル付与などを起点にワークフローが動くのでこのタイミングで過去のコメントの図が最新の状態に更新されます。


```yml
name: Comment gid

on:
  workflow_dispatch:
  issues:
    types: [opened, edited, deleted, closed, reopened, labeled] # お好みで変えると良さそう

concurrency: # 1並列で動かせれば十分なのでおまじないを追加
  group: single
  cancel-in-progress: true

jobs:
  grasp_issue:
    runs-on: ubuntu-latest
    name: Grasp issue dependencies
    steps:
      - uses: actions/checkout@v3
      - uses: k-nasa/gid@main
        with:
          label: 'root' # ここで指定したラベルが付いているものをrootとして関係を可視化します
          github_token: ${{secrets.GITHUB_TOKEN}}
```


**この図は現状PCでしか見れないことに注意が必要です。**

このツールの出力は[mermaid](https://mermaid-js.github.io/mermaid/#/)形式になっています。GitHubがmermaidをサポートしてくれたおかげで実装コスト少なく作れたのですが、モバイルだとmermaidが解釈されません。

https://github.blog/2022-02-14-include-diagrams-markdown-files-mermaid


mermaidテキストではなく画像を出力することで解決できそうなので今後改善していきたいと思います。

## まとめ

- TrackedIssueはいいぞ
- Issueの関係を可視化出来るぞ。Tracked Issue限定ではあるが (> _ <)
- 今後
  - mermaidに依存しないようにしていきたいね
  - TrackedIssueだけでなくissueのリンクやタスクリストもいい感じに表示したいね


https://github.com/k-nasa/gid
