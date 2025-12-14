---
layout: post
title: "Google Workspace DLPを触ってみた"
description: "Google Workspace DLP の社内展開を通じて得られたTipsなどをまとめた記事"
banner: 
    image: "/assets/images/2022-12-05-advent-calendar-cover.jpg"
    heading_style: "display: none"
    subheading_style: "display: none"
date: 2022-12-05
categories: IT
tag: 
- Information Security
- Google Workspace
- Google BeyondCorp
---

皆さんこんにちは、READYFOR でコーポレートエンジニアを担当している若林です。

この記事は [READYFOR Advent Calendar 2022](https://qiita.com/advent-calendar/2022/readyfor) 
5日目の記事になります。

## これはなに

READYFORでは全社の情報セキュリティ施策を進めていることが多い人間なのですが、今回は以前紹介させて頂いた[Google BeyondCorp](https://tech.readyfor.jp/entry/2020/12/07/105621)を支える要素技術のうち、現在進行形で社内展開を進めている
[Google Workspace DLP](https://support.google.com/a/answer/9646351?hl=ja)
について簡単に紹介させて頂こうと思います。

## DLP (Data Loss Prevention) ってなんだっけ　

[Wikipedia](https://en.wikipedia.org/wiki/Data_loss_prevention_software)には冒頭、以下のような説明があります。ざっくり日本語で要約すると、「情報漏洩防止のための仕組み」ということになるでしょうか。

> Data loss prevention (DLP) software detects potential data breaches/data
> ex-filtration transmissions and prevents them by monitoring,detecting and
> blocking sensitive data while in use (endpoint actions), in motion (network
> traffic), and at rest (data storage).

ただ、この表現だと抽象的すぎて情報セキュリティ全般の話に感じてしまうのではないでしょうか。

[Wikipedia](https://en.wikipedia.org/wiki/Data_loss_prevention_software)
でも補足されていますが、現在 **DLP**
と表現する場合、データに対するアクションに関して、機械学習などを通じて動的にアクティビティリスクを分析し、一定の閾値を超えた場合にそのアクションに対する警告やブロックを行うソリューションのことを指すことが一般的です。

[CIS Controls v8](https://www.cisecurity.org/controls/v8)でもIG3に分類されているように、比較的高度なセキュリティソリューションに位置付けられています。

## DLPソリューションの市場 と Google Workspace DLP の立ち位置

世の中には、様々なレイヤで動作するDLPソリューションが提供されています。
以下の表に、動作レイヤごとの代表的なDLPソリューションとその特徴をまとめてみました。

| レイヤ | サービス例 | 特徴 |
| --- | --- | --- |
| ネットワーク | [Netskope Data Loss Prevention](https://www.netskope.com/jp/products/data-loss-prevention)<br> [Zscaler Data Loss Prevention](https://www.zscaler.jp/technology/data-loss-prevention)<br>[Microsoft Defender for Cloud Apps](https://learn.microsoft.com/ja-jp/microsoft-365/compliance/dlp-use-policies-non-microsoft-cloud-apps) | 複数のSaaSを横断して同一のポリシーを適用可能。<br>フォーワードプロキシやリバースプロキシ、API型等の実装方式が存在し、できることが大きく異なる。 |
| エンドポイント | [Microsoft Endpoint DLP](https://learn.microsoft.com/ja-jp/microsoft-365/compliance/endpoint-dlp-using)<br>[BeyondCorp Threat and Data Protection を使用してデータ損失防止（DLP）機能を Chrome に統合する](https://support.google.com/a/answer/10104358?hl=ja#) | 複数のSaaSを横断してポリシー適用できるが、動作対象外のブラウザやプロファイルを利用されると制御しきれないことも。 | 
| SaaS内のデータ | [Microsoft Purview](https://learn.microsoft.com/ja-jp/microsoft-365/compliance/dlp-learn-about-dlp)<br>[Workspace の DLP を使用してデータの損失を防止する](https://support.google.com/a/answer/9646351?hl=ja) | SaaSに組み込まれている機能のため、機密表示（ラベル）等と連携した細かな制御が実現可能。 |
| データベース・ストレージ | [Amazon Macie](https://aws.amazon.com/jp/macie/)<br>[Cloud Data Loss Prevention](https://cloud.google.com/dlp?hl=ja) | 自社開発プロダクトが保有するデータに適用したい場合はこちら |

CASB/SWGにカテゴライズされるサービスは、ネットワークレイヤでのDLPソリューションを提供しており、SaaS横断でDLPポリシーを適用し制御することが可能です。
最も広範囲にDLPポリシーを適用可能なアプローチですが、使いこなすための運用工数の増加やソリューションのコストも高額になりがちです。
フォワードプロキシ型やリバースプロキシ型、さらにはSaaSとのAPI連携による制御も提供されているサービスもありますが、導入形態によってできることが大きく異なるので注意が必要です。

![netskope-pattern]({{ "/assets/images/2022-12-05-netskope-pattern.jpeg" | relative_url }})

エンドポイントにおけるDLPはブラウザの機能を使って実現していることが多いので、対象外のブラウザやSlackのような専用アプリを無効化する仕組みと組み合わせて使わないと片手落ちになりがちという欠点を抱えています。しかし導入ハードルは低いため、抜け道は一旦置いておいて事故を防ぎたいという観点では悪くない選択肢だと思います。

そして今回紹介する Google Workspace DLP は、**Google Drive に格納されているデータに特化** されているDLPソリューションであり、機密表示などのファイル分類には強いですが、SaaS横断でDLP制御できる類のソリューションではありません。
適用範囲が狭いので、情報の格納先が Google Drive に寄っていないとあまり効果を発揮しませんが、Google Drive がメインの組織であれば、追加コストを抑えつつそれなりのリスク低減に寄与することが見込めるアプローチとなっています。

## Google Workspace DLP の良いところ、改善して欲しいところ

Google Workspace DLP は、Google Workspace の [Enterprise プラン](https://workspace.google.co.jp/intl/ja/pricing.html)から利用可能な機能です。決して安いプランではないので、導入に際しては以下にまとめた良いところ・悪いところも参考にしながら、プランアップグレードの参考にして頂くと良いかもしれません。

なお、単純な設定方法に関しては、[Google公式のサポート記事](https://support.google.com/a/answer/9646351?hl=ja)に記載があるのでこの記事では特に言及しません。

### 良いところ

#### 1. ファイルに含まれる情報の機密性に応じてGoogle ドライブのラベルを自動的に付与することができる

昨年末に一般提供された[ドライブのラベル](https://workspaceupdates-ja.googleblog.com/2021/12/google-dlp.html)を、ファイルに格納されている情報に基づいて自動的に適用することができます。

従来から一般的に推奨されていた、ファイルごとの機密区分や表示運用は人が手動でやることが前提となっており、相当しっかりと教育された組織でない限り運用が追いつかず、形骸化してしまうケースが多かったのではないでしょうか。

機密表示は重要と理解しつつも、現実的にはシステムの動作に影響を与えないことから、なかなか運用に馴染まなかった実態を大きく改善できると考えております。

#### 2. ドライブやフォルダの構造に影響を与えることなく横断的にセキュリティポリシーを適用できる

組織によっては外部共有運用をしっかりと行うために、以下のような設定を駆使して外部共有が可能な共有ドライブを限定するような運用を行なっている組織も多いのではないでしょうか。

- [特定のフォルダ内のコンテンツのみ外部共有を許可する](https://support.google.com/a/answer/60781?hl=ja#zippy=%2C%E7%89%B9%E5%AE%9A%E3%81%AE%E3%83%95%E3%82%A9%E3%83%AB%E3%83%80%E5%86%85%E3%81%AE%E3%82%B3%E3%83%B3%E3%83%86%E3%83%B3%E3%83%84%E3%81%AE%E3%81%BF%E5%A4%96%E9%83%A8%E5%85%B1%E6%9C%89%E3%82%92%E8%A8%B1%E5%8F%AF%E3%81%99%E3%82%8B)

情報セキュリティ担当者としては非常に統制が効いた状態で安心できると感じるものの、本来、情報のディレクトリ構造は外部共有するかしないかという軸で分類したいわけではないと思うので、そこがセキュリティの都合だけで分断されてしまうのは運用上のマイナスが大きすぎると考えていました。

Google Workspace DLP であれば、メタデータであるラベルとファイルに含まれる文字列を元にパターンマッチングして動的に外部共有に対する制御を行うことができるので、外部共有するためだけにファイルを移動させるといった運用を回避することができます。

### 改善して欲しいところ

#### 1. リンク共有とメールアドレスを指定しての外部共有に対する操作を分離できない

Google Drive の共有機能には、大きく `リンク共有` と `メールアドレスを指定しての共有` がありますが、**DLPの動作制御においては同じ「外部共有」として扱われてしまいます。**

`リンク共有` と `メールアドレスを指定しての共有` は確かにどちらも外部共有ですが、そのリスクは全く異なるものだと思うので分離して設定できるようにして頂きたいと感じています。（例えば `リンク共有` はブロック、`メールアドレスを指定しての共有` は警告のような動作制御を行いたい）

![dlp-operation]({{ "/assets/images/2022-12-05-dlp-operation.png" | relative_url }})


#### 2. フォルダ単位でラベルが適用できない

前述したように、ファイルに格納されている情報を元にファイル単位で自動分類して頂けるのは非常にありがたいのですが、DLPにおける自動分類は一定の割合で誤った分類が発生してしまいます。

- [Data Loss Prevention Demo](https://cloud.google.com/dlp/demo/#!/?hl=ja)

[定義済みのコンテンツ](https://support.google.com/a/answer/7047475)などにおいても、分類精度が100%にならないこと自体は現時点でのテクノロジーの限界なので仕方がないと思うのですが、その分類ミス等を手動運用でカバーしていく上で、フォルダ単位で一括ラベル設定できないのが運用上手間になります。

フォルダにラベルが設定された場合、そのフォルダ配下にあるファイルに一律同じラベルを適用（それより強いラベルの場合は上書きしない）するような動作があると、より運用しやすくなると感じております。

## まとめ

今回、Google Workspace DLP の紹介をさせて頂きましたが、実はこの機能[6月末のアップデート](https://workspaceupdates-ja.googleblog.com/2022/06/google_30.html)まで、ファイルアップロードの質問を含む外部のフォームにアクセスできなくなるという致命的な業務影響があったので、展開したくてもできないものだったのです。
記事の途中で紹介させて頂いた[ドライブのラベル](https://workspaceupdates-ja.googleblog.com/2021/12/google-dlp.html)機能とともに晴れて実運用できる状態になったようなので、改めて本番展開を進めることにしたという経緯があります。

Google Workspace に対するDLPソリューションは、今回紹介した Google Workspace DLPの他に、[メールに対するDLP](https://support.google.com/a/answer/6280516?hl=ja) や [Chromeに対するDLP](https://support.google.com/a/answer/10104358?hl=ja)、さらには GCP の [Cloud DLP](https://cloud.google.com/dlp?hl=ja)、サードパーティ製品のDLPソリューションなど、非常に多くの選択肢が存在しますが、自社の環境に合ったソリューションをうまく導入することで、従業員の運用負荷をあまり増加させることなく、リスクを軽減していけると思います。

ぜひお試しあれ！