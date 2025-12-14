---
layout: post
title:  "BigQuery 生成AI関数を dbt パイプラインに組み込む際に色々詰まった話"
description: "データ基盤を構成する BigQuery × dbt パイプラインに生成AIの処理を組み込む際の試行錯誤と解決策"
banner: 
   image: "/assets/images/2025-12-15-advent-calendar-cover.png"
   heading_style: "display: none"
   subheading_style: "display: none"
date:   2025-12-15
categories: IT
tag: 
- dbt
- BigQuery
- Generative AI
---

この記事は [READYFOR Advent Calendar 2025](https://qiita.com/advent-calendar/2025/readyfor) 15日目の記事になります。

## はじめに
皆さんこんにちは。READYFOR で執行役員 VP of Productivity & Security を担当している若林です。
従来から社内ITやセキュリティ、生産性向上に関する取り組みを行っていますが、今回は昨年の Advent Calendar で記事にした[データ品質向上の取り組み](https://blog.t-wakabayashi.com/it/2024/12/12/data-quality-management.html)に続いて、データエンジニアリング寄りの話題について書いてみたいと思います。
（生成AIの盛り上がりに伴い、昨年からデータ基盤への投下時間が増えております）

## 非構造化データ活用の広がりと、データ基盤で処理する必要性
生成AIの進化により、フリーテキスト・画像・動画・音声といった非構造化データも、誰もが容易にシステムで扱える時代になりました。
従来は機械学習モデルの構築ハードルが高く、データを保存していても活用が難しかった非構造化データが、一気に処理対象になり始めています。

弊社内でも、[生成AIの熱狂に向き合った1年と社内活用推進の振り返り](https://blog.t-wakabayashi.com/it/2023/12/10/corporate-engineering-with-generative-ai.html)で紹介したような取り組みから始まり、その後 Google Workspace で利用できる Gemini アプリや NotebookLM、[スプレッドシートの AI関数](https://workspaceupdates.googleblog.com/2025/06/generate-data-with-gemini-in-google-sheets.html)等のリリースもあって、個人レベルの生産性向上に対する活用は一般化しました。

とはいえ組織全体でより大きな効果を生み出す、個々人が都度手元で処理するのではなく、共通目的の処理を横断的に集約し、前処理済みデータを配布することで、出力品質と処理効率を高める必要があります。
（生成 AI はプロンプトや入出力フォーマットを統一しないと出力が不安定になり、都度処理するのはコスト面でも不利です）

これらを踏まえると、データ基盤側で処理してしまおうと考えるのは自然な流れだと思います。

## BigQuery 生成AI関数とは
従来から ML/AI 領域のトップランナーとして君臨している Google は、BigQuery 上で直接機械学習モデルの作成・推論・評価することを支援する [BigQuery ML (BQML)](https://docs.cloud.google.com/bigquery/docs/bqml-introduction?hl=ja) を提供してきているようでした。

そして直近は、弊社のように独自モデルを構築するのではなく、基礎モデルを使うだけのニーズをカバーするため、BigQuery 上の[生成AI関数](https://docs.cloud.google.com/bigquery/docs/generative-ai-overview?hl=ja) という形で、更に組み込みやすい関数が提供され始めています。  
※ `ML.GENERATE_TEXT` と `ML.GENERATE_EMBEDDING` は、`AI.GENERATE_TEXT` と `AI.GENERATE_EMBEDDING` という関数に置き換わりつつあります

弊社では従来から BigQuery × dbt の構成でデータ基盤を構築しているので、BigQuery や dbt の標準機能でやりたいことができるなら、それに越したことはない状況でした。BigQuery の生成AI関数を使えば SQL だけで完結するため、「前処理 → 推論 → 後処理」を一つの DAGとして管理できるのは非常に魅力的です。

しかし、いざ dbt パイプライン に生成AI関数を組み込もうとすると、一筋縄ではいかない「詰まりポイント」がいくつかありましたので、紹介させて頂きます。

## 直面した課題
### 1. コスト管理
これは想像に難くないと思いますが、BigQuery の AI関数で外部の LLM を呼び出す処理は、当然ながら通常のクエリ課金とは異なる規模のコストが発生します。
一般的な規模のデータに対する dbtパイプラインであれば、最初からそこまで意識しなくても問題ない処理効率を、最初からかなり意識せざるを得ないことになります。

**具体的な対策:**
- AI関数を呼び出すモデルは必ず[incremental モデル](https://docs.getdbt.com/docs/build/incremental-models)で構築することで処理対象となるレコード数も最小限になるように設計する（もちろん可能な限り WHERE句 やハッシュ比較によるフィルタも行う）
- 普通に `full_refresh` オプションを指定するだけでは再構築（全件処理）されないようにする（[full_refresh = false](https://docs.getdbt.com/reference/resource-configs/full_refresh)）
- AI関数を呼び出す回数を可能な限り集約する。特に [AI.GENERATE_TABLE](https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-generate-table?hl=ja) を活用すると指定した形式での構造化データを出力できるので、同一入力値に対して複数の出力が必要な場合に処理はまとめることを検討する
- 開発環境では、sample データのみを使用するよう `target.name` で分岐を入れる（後述するタイムアウト問題に対処するために、本番・開発環境をAI関数専用に分離しても良さそうです）

### 2. クォータ制限とタイムアウト
`AI.GENERATE_TEXT` などの AI関数を使って大量のテキスト処理を行う場合、Vertex AI 側の API クォータに引っかかることがありました。
クォータの考え方について調べてみると、[動的共有割り当て](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/dynamic-shared-quota?hl=ja)という仕組みが採用されているようで、特に全件洗い替えを行うと数万件のリクエストが一気に飛び、`Resource Exhausted（429）` エラーが発生するだけでなく、リソースプールからの割り当てに時間がかかるようになってしまいます。（結果として、dbt のタイムアウトに引っかかるケースも）

**具体的な対策:**
- [プロビジョンドスループット](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput/overview?hl=ja)を購入する（結構高いです）
- [グローバルエンドポイント](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/locations?hl=ja#global-endpoint)を使用する（データセットと同一リージョンである必要があります）
- バッチサイズを制限する（SQL 内で `LIMIT` をかけたり、処理を分割する）

### 3. セマンティック検索の呼び出し元と前処理の共通化
セマンティック検索を実現するために、テキストからベクトルを生成するのは `AI.GENERATE_EMBEDDING` を使えば容易に実現できます。
ただ、実運用に耐える検索精度を実現するには、いきなり生のテキストをベクトル化する前に少し前処理を行うケースが多いのではないでしょうか。

その前処理も dbt パイプラインの中で実現することもできるのですが、当然ながら検索元プログラムにおいては、検索元のデータに対しても同じ前処理を行う必要があるかと思います。（データ基盤側で作成されたベクトルデータと同じ条件でベクトル化し、コサイン類似度を計算するため）

この処理をデータ基盤側と呼び出し元となるプログラム側で共通化して管理するのが難しい状態になりました。UDF化すれば外部から呼び出せるかなと思ったのですが、AI関数を含む形でUDFを作成することはできないと分かり、現状は2重管理を脱するアイデアが見つかっていません。

### 4. JSONエラーへの対応
「1. コスト管理」で構造化出力に利用できる `AI.GENERATE_TABLE` というAI関数を紹介させて頂きましたが、この `AI.GENERATE_TABLE` はLLMからの出力をテーブルに変換する際に内部で JSON をパースしています。そのJSONパース処理がエラーになるケースがあり、対処が非常に難しいです。

JSONパース自体が、`AI.GENERATE_TABLE` の内部処理に隠ぺいされていることもあり、エラーハンドリングで対処することは難しいです。
現状だと「出力が安定する高度なモデルを利用する」か、「長文を含むものや複雑すぎる構造化出力は分離する」といった場当たり的な対処しかできていない状況です。
（どちらも代わりにコストが犠牲になる対処になります）

## まとめ
今回記載させて頂いた内容は、従来から ML/AI の処理を活用していた組織は長らく向き合ってきている問題かと思いますが、我々のように従来は単なるデータ変換処理のみを dbt パイプラインで行っていた組織にとって、新たにBigQuery のAI関数 を dbt パイプラインに組み込むことは、単なるデータ変換とは異なる視点での考慮が必要になるということを改めて思い知らされました。

今回の知見が、同じような構成を検討されている方の参考になれば幸いです。

---

明日の [READYFOR Advent Calendar 2025](https://qiita.com/advent-calendar/2025/readyfor) 16日目は、[@kotarella1110](https://qiita.com/kotarella1110) の記事です。お楽しみに！
