---
layout: post
title:  "Google BeyondCorp Remote Access で実現するゼロトラストの今とこれから"
description: "READYFORに入社してから全社のゼロトラストセキュリティ基盤として導入を進めている Google BeyondCorp Remote Access について、現時点で実現できる事などをまとめてみました"
date:   2020-12-07
categories: IT
tag: 
- Google BeyondCorp
- Google Workspace
- Information Security
---

## 概要
今回はREADYFORに入社してから全社のゼロトラストセキュリティ基盤として導入を進めている Google BeyondCorp Remote Access について、現時点で実現できる事などを紹介させて頂ければと思います。

※この記事は [READYFOR Advent Calendar で掲載した記事](https://tech.readyfor.jp/entry/2020/12/07/105621)を、本ブログ用に内容調整したものになります。

## ゼロトラスト と Google BeyondCorp Remote Access
コロナウイルスでリモートワークが急速に普及したことも影響し、情報セキュリティ界隈ではここ1年の間に **「ゼロトラスト」** という用語をよく目にする様になりましたね。
ただ実際にゼロトラストモデルが Forrester の John Kindervag に考案されたのは2010年とされており、考え方自体は決して新しいものではないです。

ちなみに、ゼロトラストモデルって何？という情報セキュリティ担当の方はこちらを一読されることをお勧めします

- [ゼロトラストネットワーク――境界防御の限界を超えるためのセキュアなシステム設計](https://www.oreilly.co.jp/books/9784873118888/)

それではなぜ昨今大きく注目を浴びる様になったのでしょうか？大きく影響している要因として以下が挙げられると私は考えています。

1. リモートワークやスマートフォンの活用によりオフィスの外で働くニーズが大きくなり、ネットワーク境界形（+VPN）のセキュリティモデルが本格的に限界を迎えていること
2. Microsoft やGoogle といった Tech Giants が自社内でゼロトラストモデルを構築し、そのノウハウを元にサービスとして提供し始めていることで実装ハードルが大きく下がっていること

前述した2010年のタイミングではあくまでもアーキテクチャモデルが考案されたに過ぎず、実際にゼロトラストモデルを実現するにあたっては必要となる大半のコンポーネントを自社で実装しなければ実現できない状態にあったため、仮にこのアーキテクチャモデルが優れたものだと考えていたとしても実際に取り組める企業が少なかったのが現実です。

この記事のタイトルにも記載している **Google BeyondCorp Remote Access** は、まさに Google がゼロトラストモデルを実現するためのコンポーネント群を、サービスとして提供しているものの総称となります。

![BeyondCorp Remote Access]({{ "/assets/images/2020-12-07-google-beyondcorp-remote-access.svg" | relative_url }})

- [BeyondCorp Remote Access](https://cloud.google.com/beyondcorp-enterprise)

なお、予めお伝えしておくと Google BeyondCorp Remote Access は今年の4月に一般公開されたばかりです。（根幹となるコンテキストアウェアアクセスは、βとして長く提供されていましたが）
Google BeyondCorp Remote Access に限った話ではないですが、このようなゼロトラストモデルを構成するための市場に出てきているサービスはまだまだ未成熟であり、何か一つを採用して導入すればゼロトラストモデルが完成するわけではないので、現時点では上手く他のサービスや製品と組み合わせることで補完していく必要があります。

## Google BeyondCorp Remote Access を使って現時点で実現できること
ここからは Google BeyondCorp Remote Access を活用してREADYFOR でゼロトラストモデルを実現していく中で感じたポイントなどを、オライリーのゼロトラスト本の章構成に沿ってお伝えできればと考えております。

### 3章 ネットワークエージェント、4章 認可の判断
[コンテキストアウェアアクセス](https://support.google.com/a/answer/9275380?hl=ja)が、ネットワークエージェントへのアクセス及び認可の判断の役割を果たします。
現時点で動的なポリシーエンジンやトラストエンジン（信用スコア）は実装されておらず、あくまでもシステム管理者が以下の要素を元に事前に設定したセキュリティポリシーを満たしているかどうかが認可基準となります。

- IP（接続元IPアドレス）
- デバイス（OSのバージョン、パスワードステータス、ディスク暗号化ステータス、会社所有端末かどうか）
- アクセス元の地域（国単位）

ユーザーに関しては、Cloud Identity による認証とポリシーの適用範囲調整に依存しています。
デバイスの認可において、現時点でマルウェア対策ソフトと連携を取れないのは不足感を感じる大きなポイントではないでしょうか。

また一般提供された当初は、コンテキストアウェアアクセスの適用範囲がGoogle Workspace 内のアプリケーション + [Cloud IAP](https://cloud.google.com/iap)で接続された自社開発サービスに制限されていましたが、9月にはSAMLアプリに対しても適用可能となったことで大きな進展がありました。

- [SAML アプリに対するコンテキストアウェア アクセスの一般提供を開始](https://gsuiteupdates-ja.googleblog.com/2020/09/saml.html)

### 5章 デバイスの信頼と信用
モバイル（ここではスマートフォン、タブレット）とパソコン（ここではデスクトップ、ラップトップ）で大きく実装状況が異なります。

#### モバイル（スマートフォン、タブレット）
Device Policy アプリをデバイスにインストールすることで、ポリシーの準拠状況を確認します。
こちらは現時点でコンテキストアウェアアクセスと連携しておらず、エンドポイントで認可制御まで行っている様です。

- [iOS で Google Device Policy アプリを使用する](https://support.google.com/a/users/answer/3521320?hl=ja)
- [Android 向け Google Apps Device Policy の概要](https://support.google.com/a/users/answer/190930?hl=ja)

なお、Android OS に関してはAndroid Enterprise による仕事用プロファイルの分離やアプリケーションの配布、プロファイル単位でのワイプなど欲しい機能が揃っている感じですが、iOSは Apple Business Manager との連携は始まったものの、BYODシナリオに対してまだまだ機能不足感が否めない状況にあります。

- [Apple Business Manager との新たな連携により、会社所有の iOS デバイスの管理がシンプルに](https://gsuiteupdates-ja.googleblog.com/2020/07/apple-business-manager-ios.html)

#### パソコン（デスクトップ、ラップトップ）
Google Chrome の拡張機能である [Endpoint Verification](https://support.google.com/a/users/answer/9018161?hl=ja) をインストールすることで、デバイスの情報をポリシーエンジンに渡して認可が行われる状況になります。
元々は、従業員にネイティブヘルパーアプリを別途インストールして貰う必要があったのですが、最新のChrome を利用していれば不要になったことで展開が非常に楽になりました。（拡張機能の配布は、元々 Google Workspace の機能で可能）

また認可制御だけでなく、Windows には[高度なデスクトップセキュリティ](https://support.google.com/a/answer/9541083?hl=ja:title)も公開されており、これを展開することで Windows ログインをGoogle 認証に置き換えたり、細かなOSの設定を配布できる様になります。但し、こちらの機能を展開するには管理者によるキッティング作業が必須な状況にあり、Apple の Device Enrollment Program や Windows Autopilot といったゼロタッチ方式の展開は現時点で難しい状況にあります。

### 6章 ユーザーの信頼と信用
従来より存在している [Cloud Identity](https://support.google.com/cloudidentity/answer/7319251?hl=ja)が、この役割を果たします。

2要素認証の強制やSAML・OIDCといった SSO、更には SCIM によるプロビジョニング等にも対応できます。
Microsoft や Okta といった IDaaS のトップベンダーと比較して、設定がきめ細やかに優しく提供されている状況ではないですが、Google Workspace に標準でついてくる機能としては非常にコスパが高い状態にあると考えています。

なお、SSO設定が難しいアプリ向けに共有パスワード用のパスワードマネージャーに近い機能も展開されていますが、現時点では設定管理を分散させることができない仕様（特権管理者のみ可能）になっており、そこが解消されれば活用の幅が広がりそうだなぁという状況にあります。

- [G Suite と Google Cloud Identity Premium でパスワードが保管されたアプリのシングル サインオンが可能に](https://gsuiteupdates-ja.googleblog.com/2019/11/g-suite-google-cloud-identity-premium.html)

### 7章 アプリケーションの信頼と信用
こちらは、Google でいうと [Cloud Code](https://cloud.google.com/code)や[Cloud Run](https://cloud.google.com/run)によって構築されるCI/CD パイプラインが担う部分になるかと思います。

現時点では、Google BeyondCorp Remote Access と強く紐づけられている領域ではないのでスキップさせて頂きます。

### 8章 トラフィックの信頼と信用
こちらも現時点では、Google BeyondCorp Remote Access と強く紐づけられている領域ではないので、別途 RADIUS や CASB 等を導入して補完していく他ない状況かと思います。

## Google BeyondCorp Remote Access の今後の動向
ここまで Google BeyondCorp Remote Access の現状をツラツラと書かせて頂きましたが、ゼロトラストモデルを構築しきろうと思うとまだまだ機能が不足している状況にあることが分かったかと思います。
Google もかなりのペースで機能リリースを進めていますが、不足している部分にはパートナーとの相互接続も進めることで機能補完を図っている様です。

- [BeyondCorp Alliance の展開によりゼロトラストの普及を促進](https://cloud.google.com/blog/ja/products/identity-security/google-cloud-announces-new-partners-in-its-beyondcorp-alliance)

総合的に見てもこの領域でライバルとなる Microsoft や Akamai に遅れをとっている感は否めないので、今後の機能開発・パートナー戦略には引き続き注目していければと考えております。