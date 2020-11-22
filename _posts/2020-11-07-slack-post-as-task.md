---
layout: post
title:  "Slack における問い合わせ対応を高度化してみた"
date:   2020-11-07
categories: IT
tags:
- Slack
- Google Apps Script
- Google Data Portal
---

## 概要
Slack で問い合わせ管理専用チャンネルを運用する際に良く直面する問題に対して、別のソリューションを入れるのではなく、GASで機能拡張することで対応してみたお話

## よくある Slack 問い合わせ対応チャンネル運用
Slack 公式にもあるような問い合わせ専用チャンネルが存在し、従業員から問い合わせが順次飛んでくる形で運用されているケースが多いかなと思います。

![ITサポート]({{ "/assets/images/2020-11-07-it-servicedesk.png" | relative_url }})

問い合わせを受け付けている側は、以下のような流れで対応を行うケースが多いと思います。

1. 問い合わせの投稿に対してスレッドで回答
   - 画像と異なりますが、基本的にスレッドでやらないと流れ過ぎます
2. 問い合わせ対応が完了したら、絵文字で終了のマークを付ける
   - これも画像ではやってないですが、ある程度のチームで対応してる場合一般的だと思われます

## 上記運用の問題点
Slackを使って社内の問い合わせを受けることで、問い合わせ側の体験が良かったり、問い合わせがオープンになることで同じ内容の問い合わせを複数の部署・従業員から受けることの抑制に繋がったりといいことも多いのですが、対応側としては以下のような問題点が顕在化してきます。

- 問い合わせの対応が長期化した際にスレッドが流れてしまい、未完了の問い合わせが分からなくなってくる（ステータス管理）
- 問い合わせ対応が重要な業務となっているにも関わらず、対応コストが可視化されにくい（定量化・分析）

## 改善内容
Slack API & Google Apps Script & Google Data Portal を使って、残タスクの一覧と問い合わせ件数のレポートが自動で更新されるダッシュボードを作成。
なお、Slackの運用ルールとしては以下

1. 問い合わせが完了した投稿に対しては、:sumi:をつける
2. 問い合わせではない投稿に対しては、:filter:をつける（件数から除外）
3. 問い合わせに対しては、スレッドでやりとりを行う（1スレッドを1件の問い合わせとしてカウント）

### 実装したダッシュボード
![Data Portal]({{ "/assets/images/2020-11-07-Slack-Data-Portal.png" | relative_url }})

### 自動更新の流れ
1. Google Apps Sciprt の [Time-driven triggers](https://developers.google.com/apps-script/guides/triggers/installable#time-driven_triggers)により、15分に一度関数がトリガーされる
2. Slack API（[conversation.history](https://api.slack.com/methods/conversations.history)）から、特定のチャンネルの履歴を全て取得し、スプレッドシートの２次元データとして展開
3. スプレッドシートの情報を元に構成されている Google Data Portal のダッシュボードが更新される（15分間隔）

### サンプルコード（Google Apps Script）

```javascript
// 変数
var token = PropertiesService.getScriptProperties().getProperty('token');
var channelid = PropertiesService.getScriptProperties().getProperty('channnelid');

// 取り扱うシート
var ss = SpreadsheetApp.getActiveSpreadsheet();
var sh = ss.getActiveSheet();

// メッセージが持つフィールドのうち出力項目を列挙
var outputField_ary = [
    "type",
    "user",
    "text",
    "client_msg_id",
    "attachments",
    "edited",
    "thread_ts",
    "reply_count",
    "replies",
    "subscribed",
    "unread_count",
    "parent_user_id",
    "ts",
    "subtype",
    "reactions"
];

// カウントしたい絵文字を列挙
var outputReaction_ary = [
    "sumi",
    "filter"
];

function main() {
    // スプレッドシート初期化
    sh.clear();

    // フィールド名設定
    sh.getRange(1, 1, 1, outputField_ary.length).setValues([outputField_ary]);
    sh.getRange(1, outputField_ary.length, 1, outputReaction_ary.length).setValues([outputReaction_ary]);

    // 取得対象期間設定
    var startdate = '2017/4/1';
    var enddate = new Date();
    start_ts = getStartTs(startdate);
    end_ts = getEndTs(enddate);

    // データ取得
    getChannelMessage(start_ts, end_ts)
}

function getStartTs(val) {
    var start_date = new Date(val);
    start_date.setHours(0);
    start_date.setMinutes(0);
    start_date.setSeconds(0);
    start_date.setMilliseconds(0);
    var start_ts = start_date.getTime() / 1000;
    return start_ts;
}

function getEndTs(val) {
    var end_date = new Date(val);
    end_date.setHours(23);
    end_date.setMinutes(59);
    end_date.setSeconds(59);
    end_date.setMilliseconds(0);
    var end_ts = end_date.getTime() / 1000;
    return end_ts;
}

function unixTime2ymd(intTime) {
    var d = new Date(intTime * 1000);
    var year = d.getFullYear();
    var month = d.getMonth() + 1;
    var day = d.getDate();
    var hour = ('0' + d.getHours()).slice(-2);
    var min = ('0' + d.getMinutes()).slice(-2);
    var sec = ('0' + d.getSeconds()).slice(-2);

    return (year + '/' + month + '/' + day + ' ' + hour + ':' + min + ':' + sec);
}

function getChannelMessage(start_ts, end_ts) {
    var url = "https://slack.com/api/conversations.history?token=" +
        token + "&" +
        "channel=" +
        channelid + "&" +
        "oldest=" +
        start_ts + "&" +
        "latest=" +
        end_ts + "&" +
        "count=1000&pretty=1";

    // 日付をもとにチャンネル内のメッセージを取得
    var response = UrlFetchApp.fetch(url);
    var jsonData = JSON.parse(response.getContentText());

    // 2次元配列
    var chathistory_ary = [];
    var message_ary = [];

    // チャット履歴の巡回
    for (var i = 0; i < jsonData.messages.length; i++) {
        // メッセージフィールドの巡回
        for (var j = 0; j < outputField_ary.length; j++) {
            // フィールドによって対応を変更
            if (!jsonData.messages[i][outputField_ary[j]]) {
                // 対応するフィールドが定義されていない場合、空欄を配列に追加
                if (outputField_ary[j] == 'reactions') {
                    for (var k = 0; k < outputReaction_ary.length; k++) {
                        message_ary.push(0);
                    }
                } else {
                    message_ary.push("");
                }
            } else if (outputField_ary[j] == 'ts' || outputField_ary[j] == 'thread_ts') {
                // タイムスタンプは、Date型に変更して配列に
                message_ary.push(unixTime2ymd(parseInt(jsonData.messages[i][outputField_ary[j]])));
            } else if (outputField_ary[j] == 'reactions') {
                // リアクションは、カウント対象のみを配列に追加
                for (var l = 0; l < outputReaction_ary.length; l++) {
                    message_ary.push(0);
                    for (var k = 0; k < jsonData.messages[i][outputField_ary[j]].length; k++) {
                        if (jsonData.messages[i][outputField_ary[j]][k]["name"] == outputReaction_ary[l]) {
                            message_ary[j + l]++;
                        }
                    }
                }
            } else {
                // その他のフィールドは、取得した値のまま配列に追加
                message_ary.push(jsonData.messages[i][outputField_ary[j]]);
            }
        }
        // チャンネルアレイにメッセージ情報を挿入
        chathistory_ary.push(message_ary);

        // メッセージ配列の初期化
        message_ary = [];
    }

    // スプレッドシートに転記
    sh.getRange(sh.getLastRow() + 1, 1, chathistory_ary.length, outputField_ary.length + outputReaction_ary.length - 1).setValues(chathistory_ary);

    //メッセージ件数を確認し、ページネーション
    if (chathistory_ary.length == 1000) {
        getChannelMessage(start_ts, jsonData.messages[999]['ts']);
    }
}
```

## 最後に
こちらを導入した結果、対応漏れを防げたり、問い合わせ件数の増加に合わせた人員リソースの調整などの説得力が増したと感じています。

上記はあくまでもフリーテキストでの投稿を想定した実装になっていますが、Slack のワークフロービルダーを使って問い合わせのフォーマットを厳格にすることで、問い合わせカテゴリ毎に分析したりすることも可能になると思います。
（ワークフロービルダーで投稿された情報をAPI経由で取得した場合、意外と扱いにくいJSON形式になっていますが・・・）

その他、アサイン管理や対応ステータス（未対応・完了以外のステータス）を追加する場合も、対応する絵文字を先に定義した上で、それを元に処理を追加することで、様々な応用が利く形になっていると思います！