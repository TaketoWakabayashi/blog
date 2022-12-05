---
layout: post
title:  "Google WorkspaceのDLP機能について考えてみた"
description: ""
date:   2022-12-05
categories: IT
---

皆さんこんにちは、READYFOR でコーポレートエンジニアを担当している若林です。
この記事は[READYFOR Advent Calendar 2022](https://qiita.com/advent-calendar/2022/readyfor) 5日目の記事になります。

## これはなに
READYFORでは全社の情報セキュリティ施策を進めていることが多い人間なのですが、今回は以前紹介させて頂いた[Google BeyondCorp](https://tech.readyfor.jp/entry/2020/12/07/105621)を支える技術要素のうち、現在進行形で社内展開を進めている Google Workspace DLP について簡単に紹介させて頂こうと思います。

## DLP (Data Loss Prevention) ってなんだっけ　

[Wikipedia](https://en.wikipedia.org/wiki/Data_loss_prevention_software)には以下のように記載があります。ざっくり日本語で要約すると、「情報漏洩防止のための仕組み」ということになるでしょうか。

>Data loss prevention (DLP) software detects potential data breaches/data ex-filtration transmissions and prevents them by monitoring,detecting and blocking sensitive data while in use (endpoint actions), in motion (network traffic), and at rest (data storage).

ただ、この表現だと抽象的すぎて情報セキュリティ全般の話に感じてしまうのではないでしょうか。

Wikipedia でも補足されていますが、現在 **DLP** と表現する場合、データに対するアクションに対して、機械学習などを通じて動的にアクティビティリスクを分析し、一定の閾値を超えた場合にそのアクションに対する警告やブロックを行うソリューションのことを指すことが一般的です。

## DLPソリューションの市場とGoogle Workspace DLPの立ち位置

世の中には、様々なレイヤで多くのDLPソリューションが提供されています。

- [Microsoft Purview](https://learn.microsoft.com/ja-jp/microsoft-365/compliance/dlp-learn-about-dlp)
- [Symantec Data Loss Prevention](https://jp.broadcom.com/products/cybersecurity/information-protection/data-loss-prevention)
- [Netskope Data Loss Prevention](https://www.netskope.com/jp/products/data-loss-prevention)
- [Zscaler Data Loss Prevention](https://www.zscaler.jp/technology/data-loss-prevention)
- [Slack の Discovery API ガイド](https://slack.com/intl/ja-jp/help/articles/360002079527)
- [Cloud Data Loss Prevention](https://cloud.google.com/dlp?hl=ja)
- [Workspace の DLP を使用してデータの損失を防止する](https://support.google.com/a/answer/9646351?hl=ja)


## Google Workspace DLPの設計で困ったこと