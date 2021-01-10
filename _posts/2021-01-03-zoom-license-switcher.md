---
layout: post
title:  "Zoom ライセンスの使いまわし運用をGASで自動化してみた"
date:   2021-01-03
categories: IT
tag: 
- Google Apps Script
- Google Calendar
- Zoom
---

## 概要
コロナの影響もあって、急遽 Zoom の有料プランを契約した会社も多いんじゃないでしょうか。

Zoom のライセンスは同時開催しない限りは以下の記事にもあるように使いまわせるようになっており、ライセンス単価がそれなりに高いこともあって、多くの企業で使い回し運用がなされていると思います。

- [【ZOOM活用】社内で複数の有料アカウントを管理する方法](https://note.com/namakemono_info/n/n9763a7e2c5cf)

※ちなみに**アカウントの共有**は禁止されており、あくまでも**ライセンスの動的割り当て**が許可されている形です。セキュリティの観点でも共有アカウント運用はリスクが大きいのでやめましょう。

しかしライセンスの使い回し運用というのは結構曲者で、ぱっと見のサービス利用料は低く抑えられるものの、割り当て依頼対応や同時開催の管理で結構な時間を取られてしまいます。

今回はそんな Zoom ライセンス運用の負の部分を、可能な限りスマートに自動化実装してみたので紹介させて頂きます。

## Zoom のライセンス形態について
まず大前提となる Zoom のライセンスについておさらいです。下記にもあるように、40分を超えるミーティングを実施したい場合、ミーティングホストが有料ライセンスを付与されている必要が出てきます。

![Zoomプラン]({{ "/assets/images/2021-01-03-zoom-plans.png" | relative_url }})

- [Zoom Pricing](https://zoom.us/pricing)

社内のミーティングに関しては基本的に Google Meet で問題ないのですが、社外に向けたウェビナー開催や Google Meet の利用が許可されていない取引先とのミーティングなどで、Zoom 利用を避けられないという現実もあり、READYFORでも別途 Zoom Pro プランを2つ契約して、ライセンスを上手く使いまわしています。

## 運用における問題点
前述した通りZoom Pro ライセンスは同時開催される分だけ必要となり、ライセンスを払い出す側としては以下のような情報を管理し、運用していく必要があります。

1. 誰がいつ Zoom ライセンスを使いたいか管理
2. 同時に開催される場合、その同時開催数がライセンス保有数を超過しないか確認
3. ミーティングが開催される時点で、ミーティングのホストにライセンスを割り当てる運用

※下記の記事にもありますが、ミーティングURLはライセンスの割り当て有無に関わらず事前発行可能で、実際にそのミーティングが始まるタイミングでライセンスが割り当てられていれば、そのミーティングに対してライセンスが反映されます。

- [基本ユーザーをライセンス済みユーザーに変更した場合、ミーティング時間は40分に制限されますか？](https://zoom-support.nissho-ele.co.jp/hc/ja/articles/360046020671)

1週間に一人使いたい程度の状況であればSlackで依頼を受けて対応する形で十分回るのですが、週に何回も依頼が来るような状況になってくると、難しくなってきます。

今回のソリューションは、実際に週に3,4回使いたいという依頼が飛んでくるようになってきたため、運用負荷の問題を解消するために、実装したものになります。

## 実装内容
Google Calendar & Google Apps Script & Zoom API を使って、Zoom の利用予約と実際のライセンス割当運用の自動化を実現してみました。

なお、Google Calendar の運用ルールとしては、以下としています。

1. Zoom Pro ライセンスを利用したいユーザーは、事前に登録されている Zoom ライセンスリソース（会議室等と同じ位置付け）を含める形で予約を行う
    - [ビルディング、設備や機能、カレンダー リソースの設定](https://support.google.com/a/answer/1033925)にて事前に登録
    - ライセンスを複数保有している場合はその数だけ登録しておくことで、同時開催数の上限を制限する
    - 一つのライセンスにつき、1日1予約まで
2. 毎日夜中、カレンダーの登録内容に基づいて次の日に予約しているユーザーにライセンスを付与する

![Zoomライセンス予約]({{ "/assets/images/2021-01-03-Zoom-Resorce.png" | relative_url }})



この 2 の動作を、GAS + Zoom API で実現した詳細が以下になります。

### 自動ライセンス割当運用の流れ
GAS で実装したアルゴリズムの概要は以下になります。

1. 毎日夜中に起動するように、[GASの時間駆動型トリガー](https://developers.google.com/apps-script/guides/triggers/installable#time-driven_triggers)を設定
2. [Calendar API](https://developers.google.com/calendar/v3/reference/calendars/get)を利用して、前日と当日の Zoom ライセンスリソースに対する予約状況を取得
3. 2で取得した予約状況を元に、[Zoom API(User Update)](https://marketplace.zoom.us/docs/api-reference/zoom-api/users/userupdate)を利用して、前日分のライセンス剥奪と当日のライセンス付与

### サンプルコード（Google Apps Script）
動作を実現するGASのサンプルコードは以下になります。

スクリプトプロパティを利用している以下の変数があるので、環境に合わせて設定が必要になります。

- Zoom ライセンスリソースのカレンダーID
- Zoom API用の認証に利用する[JWT](https://marketplace.zoom.us/docs/guides/auth/jwt)取得用`API Key` と `Secret`
- Slack 通知用の `INCOMING WEBHOOK`

```javascript
const ZOOM_CALENDER_IDS = [
  PropertiesService.getScriptProperties().getProperty('ZOOM_CALENDAR_ID_1'),
  PropertiesService.getScriptProperties().getProperty('ZOOM_CALENDAR_ID_2')'
]

function main() {
  const today = new Date();
  const tomorrow = new Date(today);
  tomorrow.setDate(tomorrow.getDate() + 1);

  let strHeader = ':zoom: *Zoomライセンス操作通知(' + Utilities.formatDate(tomorrow, 'JST', 'yyyy/MM/dd') + ')* :zoom:\n';
  let strBody = ``;
  try {
    const token = getZoomToken();

    for (let i = 0; i < ZOOM_CALENDER_IDS.length; i++) {
      let cal = CalendarApp.getCalendarById(ZOOM_CALENDER_IDS[i]);
      let todayEvents = cal.getEventsForDay(today);
      let tomorrowEvents = cal.getEventsForDay(tomorrow);

      if (!todayEvents.length && !tomorrowEvents.length) {
        continue;
      }
      strBody += cal.getTitle() + '\n';
      if (todayEvents.length) {
        strBody += '剥奪：' + getEventsStr(todayEvents) + '\n';
        revokeZoomMeetingLicense(token, todayEvents[todayEvents.length - 1].getCreators()[0]);
      }
      if (tomorrowEvents.length) {
        strBody += '付与：' + getEventsStr(tomorrowEvents) + '\n';
        grantZoomMeetingLicense(token, tomorrowEvents[0].getCreators()[0]);
      }
      if (todayEvents.length > 1 || tomorrowEvents.length > 1) {
        strBody += ':warning: <@taketo.wakabayashi> 特殊ケースのため、ライセンス処理が正しく行われているか確認してください' + '\n';
      }
    }

    if (strBody) {
      postToSlack_(strHeader + strBody);
    }
  } catch (e) {
    postToSlack_(':warning: <@taketo.wakabayashi> エラーが発生したようなので、状況を確認してください');
  }
}

/* 時刻の表記をHH:mmに変更 */
function _HHmm(str) {
  return Utilities.formatDate(str, 'JST', 'HH:mm');
}

/**
* Slackにメッセージを投稿します。
* @param {string} message - メッセージ
*/
function postToSlack_(message) {
    var params = {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        payload: '{"text":"' + message + '"}'
    };
    UrlFetchApp.fetch(PropertiesService.getScriptProperties().getProperty('WEBHOOK_URL'), params);
}

function getEventsStr(events) {
  var eventStr = '';
  for (let i = 0; i < events.length; i++) {
    eventStr += _HHmm(events[i].getStartTime()) + '~' + _HHmm(events[i].getEndTime()) + '　' + events[i].getTitle() + ' at:<@' + events[i].getCreators()[0].replace('@readyfor.jp', '') + '>\n';
  }
  return eventStr;
}

function getZoomToken() {
    const apiKey = PropertiesService.getScriptProperties().getProperty('API_KEY');
    const apiSecret = PropertiesService.getScriptProperties().getProperty('API_SECRET');
    const header = Utilities.base64Encode(JSON.stringify({
        'alg': 'HS256',
        'typ': 'JWT'
    }));

    const claimSet = JSON.stringify({
        "iss": apiKey,
        "exp": Date.now() + 3600
    });

    const encodeText = header + "." + Utilities.base64Encode(claimSet);
    const signature = Utilities.computeHmacSha256Signature(encodeText, apiSecret);
    const jwtToken = encodeText + "." + Utilities.base64Encode(signature);
    return jwtToken;
}

function grantZoomMeetingLicense(token, userId) {
    var data = {
        'type': 2
    };
    var options = {
        'method': 'patch',
        'contentType': 'application/json',
        'headers': { 'Authorization': 'Bearer ' + token },
        // Convert the JavaScript object to a JSON string.
        'payload': JSON.stringify(data)
    };
    UrlFetchApp.fetch('https://api.zoom.us/v2/users/' + userId, options);
}

function revokeZoomMeetingLicense(token, userId) {
    var data = {
        'type': 1
    };
    var options = {
        'method': 'patch',
        'contentType': 'application/json',
        'headers': { 'Authorization': 'Bearer ' + token },
        // Convert the JavaScript object to a JSON string.
        'payload': JSON.stringify(data)
    };
    UrlFetchApp.fetch('https://api.zoom.us/v2/users/' + userId, options);
}
```

## 終わりに
今回は、 Zoom のライセンス使いまわし運用を楽にする方法を紹介させて頂きました。

今回紹介した運用方式だと毎日夜中に一度だけ切り替え処理が行われますが、日中にライセンス割り当てを切り替えたい場合などは、今回の実装方式だとカバーできないシナリオになるので、人がカバーする必要が出てきますのでお気をつけください。（コードの中でも、1日二つ以上の予約が入っている場合は警告するようになっています）

上記のような制約はあるものの、Zoom はProライセンスでも ¥2,000/Monthly/license と高額なので、Google Workspace を導入しているような企業では、別途全従業員のライセンスを手配するのは割りに合わないと感じるケースが多いかと思います。
そういった際に今回紹介させて頂いた方法で運用コストを大きく低減できると思いますので、是非役立ててみてください！