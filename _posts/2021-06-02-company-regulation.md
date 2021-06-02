---
layout: post
title:  "Github & Jekyll & GAE & IAP で規程更新管理・社内公開プラットフォームを構築してみた"
description: "Github"
date:   2021-06-02
categories: MISC
tag: 
- Github
- Jekyll
- Google App Engine
- Identity-Aware Proxy
---

## 概要
今や世界中のソフトウェア開発プラットフォームとして利用されている Github ですが、その有用性はプログラムコードの管理にとどまらず、厳密な更新管理が必要となる法令や社内規程、更にはサービス等のドキュメントにも活用が広がっています。

- [法律をGitHubのプルリクエスト機能を使って修正するその一部始終が公開中](https://gigazine.net/news/20190202-law-github-pull-request/)
- [Markdown と GitHub で社内規程を便利に管理](https://techlife.cookpad.com/entry/2019/06/26/182322 )
- [Microsoft Docs 共同作成者ガイド概要](https://docs.microsoft.com/ja-jp/contribute/)

今回は、そんな世の中の流れを受けて社内規程の管理をGithubにのせてみたので、その取り組みや実装についてご紹介させて頂きます。

ちなみに作成した公開サイトのイメージはこちら

![READYFOR社内規程]({{ "/assets/images/2021-06-02-company-regulation.jpg" | relative_url }})

## 一般的な規程更新・管理プロセスの問題
下記クックパッドさんの記事にもある通りだと思うので、こちらを参照ください。

- [Markdown と GitHub で社内規程を便利に管理](https://techlife.cookpad.com/entry/2019/06/26/182322 )



## 規程管理の要件・運用
規程管理プラットフォームに求める要件と運用イメージを言語化したものを順番に紹介させて頂きます。

### 業務要件・機能対応表
規程管理の業務要件と、実装したプラットフォームの機能対応表は以下になります。


No | 内容 | 対応する機能
--------- | --------- | ---------
1 | 規程フォーマットの文章を管理できること | ・マークダウン形式で管理すれば可能<br>(見出し、番号付きリスト、箇条書き、表) 
2 | 変更すべき内容をバックログ的に蓄積できること | ・Github Issue を使えば可能
3 | レビュー・決裁運用が容易であること | ・Github Pull Request におけるレビュー機能を使えば可能<br>・CodeOwnerを設定することで、承認を強制することも可能
4 | 変更履歴を容易に追跡可能であること | ・Github の `master` ブランチを直接いじらず、Github Pull Request を使えば可能
5 | 複数の変更が同時に走る場合にも対応できること | ・Github branch を利用することで確認可能
6 | 社内に限定したアクセス制御ができること | ・GAE + IAPで対応可能
7 | 規程を横断して検索可能であること | ・rundocs/jekyll-rtd-theme で対応可能

なお、弊社独自の要件として全従業員がGithub アカウントを持たない状態で、従業員のみにアクセス許可をしたいというものがありました。
この要件は、元々開発のメンバーのみがGithubアカウントを保有していた状態だったこともあり、ROIを考えるとこのためだけに全従業員へのGithubアカウントを用意することは難しかったという背景からきています。

### 規程改定運用フロー
Github上での規程運用の流れは以下になります。なお、規程改定に参加するメンバーはCUIに慣れていないことが多いので、実際に以下の流れは全てGithub のGUIから行っています。

1. 規程の記載内容と運用実態の乖離に気づいた場合や先々の法改正等、すぐに修正する必要はないがバックログとして積んでおきたい課題がある場合に、その課題を説明した Issue を作成（すぐに修正に着手しなくても良い）
2. 特定の目的や課題を背景に規程更新PJを行う必要がある場合は、その目的や課題を説明した Issue を作成し、対応担当者をアサイン（1. で溜まっているバックログを規程に反映するプロジェクトを走らせる場合は、アサインからスタート）
3. 担当者は 更新用のbranchを作成し、目的に沿った修正内容を 更新branch に Commit していく
4. 完成したら Pull Request を作成し、レビュアーにレビュー依頼を投げる
5. レビュアーは変更内容をレビューし、問題ないようであれば CodeOwner に規程改訂における承認プロセスの実行を依頼する
6. CodeOwner は規程改訂の決議を取得した上で、Pull Request の承認を行う（施行日までは、マージしない）
7. 施行日を迎えたタイミングでマージを実施する

### 規程フォーマット（Markdown）
規程でよく使われるフォーマットを Markdown に落とした場合のルールは以下にしました。若干 Word で管理していたものと見栄えが変わる部分もありましたが、Github 上での管理・編集のしやすさを優先し、無駄なものは削ぎ落とす方針で決めました。

項目 | 利用する機能 | 書き方
:-: | --- | --- 
表題（タイトル） | 見出し1 | #
章 | 見出し2 | ##
節 | 見出し3 | ###
条 | （見出し4） | #### （○○○）
項 | 番号付きリスト（第一階層） | 1.
号 | 番号付きリスト（第二階層） | 1.（タブでずらす）
附則 | Pull Request 側で表現し、本文からは削除 | -

## 規程管理プラットフォームの実装
上記要件を満たすプラットフォームの実装を大まかに紹介させて頂きます。
なお、主な構成要素は以下になります。

- 静的サイトジェネレータ：[Jekyll](http://jekyllrb-ja.github.io/) & [rundocs/jekyll-rtd-theme](https://github.com/rundocs/jekyll-rtd-theme)
- サイトホスティング・認証/認可：[Google App Engine](https://cloud.google.com/appengine?hl=ja) & [Identity-Aware Proxy](https://cloud.google.com/iap)
- CI/CD：Github Actions

### 静的サイトジェネレータ：[Jekyll](http://jekyllrb-ja.github.io/) & [rundocs/jekyll-rtd-theme](https://github.com/rundocs/jekyll-rtd-theme)
[認証/認可及びコスト（全従業員分のアカウント発行が難しい）](https://github.blog/jp/2021-01-25-access-control-for-github-page/)の関係で Github Pages をそのまま利用することは難しかったので、別途規程公開用の静的サイトジェネレータを利用しました。Jekyllを選択した理由は、シンプルにGithub Pages が正式にサポートしているからですね。（このブログをやっていたこともあって利用も慣れているので）

一般的にこのブログに使われているようなブログ用のテーマが多いのですが、ドキュメント管理用のテーマも開発されていたりして、今回は見た目 & 管理の容易さから[rundocs/jekyll-rtd-theme](https://github.com/rundocs/jekyll-rtd-theme)を選択しました。  
※但し、このテーマはGithub Pages でホスティングされることを前提に作られていることもあって、ちょくちょくカスタマイズしないと綺麗にいかなかったです。

### サイトホスティング・認証/認可：[Google App Engine](https://cloud.google.com/appengine?hl=ja) & [Identity-Aware Proxy](https://cloud.google.com/iap)
静的サイトホスティングはどこでも良かったのですが、従業員の認証認可は Google Workspace のアカウントで行いたいという思いがありました。

元々、[Google BeyondCorp Remote Access で実現するゼロトラストの今とこれから](https://blog.t-wakabayashi.com/it/2020/12/07/google-beyondcorp-remote-access.html)でも紹介しているように、Google を基盤に据えたゼロトラストモデルを構築していることもあったので、Identity-Aware Proxy の活用が用意な Google App Engine でホストすることにしました。

### CI/CD：[Github Actions](https://github.co.jp/features/actions)
Github Pages であればここもよしなにやってくれるのですが、今回はGAE上にデプロイする必要がある構成になってしまっているので、Github Actions を利用してGAEにデプロイしました。
[こちら](https://qiita.com/nakano-shingo/items/a8d4a6cf456160d8a3ed)の記事が非常に参考になりました。

## 終わりに
今回は社内の規程管理を効率化した取り組みの全体像を紹介させて頂きました。（技術的な部分は省略気味ですいません）

実際にこういった取り組みを進める際には、社内の関係者を巻き込んで進めるところが一番大変だったりしますが、READYFORではこういった業務プロセスの改善には前のめりに参加してくれる人が多いので助かっております。

最近では、経産省がGithubでドキュメントを公開する動きがあるなど、行政機関でも活用が伺える Github。是非一度お試しあれ