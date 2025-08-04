<img src="/images/yoro_bot_400x400.png" alt="yoro_bot icon image" width="100" height="100" />

### 更新履歴

- 2025/06/27 ー 『日本人の身体観の歴史』からの引用をツイートデータベースに追加
- 2025/06/05 ー wikipediaリプライ機能を実装
- 2025/01/09 ー botのつくり方を追記
- 2025/01/06 ー GitHubPagesにこのページを作成
- 2024/12/22 ー GoogleAppsScriptに移植して稼働再開
- 2016/01/23 ー follower数が1000人を超えました（しばらく前に）
- 2015/03/10 ー follower数が700人を超えました
- 2015/01/23 ー follower数が600人を超えました
- 2014/12/12 ー follower数が500人を超えました
- 2014/09/23 ー ver.8へのアップデート（Viewer機能搭載に伴う全面的なコードの見直し）
- 2014/08/21 ー follower数が300人を超えました
- 2014/05/17 ー 『からだを読む』からの引用をツイートデータベースに追加
- 2014/05/02 ー follower数が100人を超えました
- 2014/02/23 ー ver.7へのアップデート（スウィープ機能を実装、webapp→webapp2、db→ndb）
- 2014/01/26 ー メインの処理を行っているPythonのコードを公開しました
- 2014/01/25 ー ver.6へのアップデート（データ保存にDataStoreを使用、それぞれの引用毎のツイート回数を記録）
- 2013/10/27 ー 本稼働開始
- 2013/10/11 ー 試験稼働開始

### 凡例

|   |   |   |
|---|---|---|
|「＼」|全角バックスラッシュ|段落改行及び単純改行のその改行位置|
|「／」|全角スラッシュ|離れた二つ以上の部分を連結した際のその分割位置|
|「（※）」|全角丸括弧内の全角コメジルシ|管理者による文脈を補足する為の註釈|

### 引用元

|   |   |   |
|---|---|---|
|済|『ヒ見方』|『ヒトの見方』|
|済|『か見方』|『からだの見方』|
|済|『脳見方』|『脳の見方』|
|済|『虫眼』|『虫眼とアニ眼』|
|済|『唯脳論』|『唯脳論』|
|済|『カミと』|『カミとヒトの解剖学』|
|済|『人科学』|『養老孟司の人間科学講義』|
|済|『文学史』|『身体の文学史』|
|済|『を読む』|『からだを読む』|
|済|『身体観』|『日本人の身体観の歴史』|
|未|『遺言。』|『遺言。』|

### botのつくり方

1. Googleスプレッドシートを作成する。
   - A列：140文字以内のツイート本文
   - B列：ツイート回数の記録、初期値はゼロ
   - C列：1から通し番号を振る
   - D列：LEN(A#)、D1に`=LEN(A1)`と入力してからD列を全選択して`Ctrl + Enter`
   - E列（任意）：空でない場合、A列のツイートへのリプライとしてぶら下がる文字列
2. 「拡張機能」からAppsScriptを作成する。
   - 「ライブラリを追加」からOAuth2ライブラリを追加
     - 1B7FSrk5Zi6L1rSxxTDgDEUsPzlukDsi4KGuTMorsTQHhGBzBkMun4iDF
   - 「プロジェクトの設定」からスクリプトIDを覚えておく。
3. TwitterDeveloperPortalでAppを作成する。
   - 「User authentication settings」の「Callback URI / Redirect URL」の欄に上で覚えたIDを含むURLを入力する。
     - https://script.google.com/macros/d/スクリプトID/usercallback
   - 「Keys and tokens」から「Client ID」と「Client Secret」を保存する。
4. AppsScriptの「エディタ」に以下の`コード.gs`をコピーして貼り付ける。
5. 「プロジェクトの設定」の「スクリプトプロパティを追加」から、保存した値を設定する。
   - CLIENT_ID
   - CLIENT_SECRET
6. 「エディタ」から`initialSetUp()`関数を選択し「実行」する。
   - ダイアログに従い、AppsScriptからGoogleアカウントへのアクセスを許可する。
   - 実行ログに表示されるURLにブラウザからアクセスし、Twitterアカウントへのアクセスを許可する。
7. 「トリガー」から必要なトリガーを追加する。
   - 30分おきのトリガーで`drawLots(48)`だと一日一ツイート平均となる。

### コード.gs

```javascript
function initialSetUp() {
  const sp = PropertiesService.getScriptProperties();
  const clid = sp.getProperty('CLIENT_ID');
  const clsec = sp.getProperty('CLIENT_SECRET');
  if (!clid || !clsec) {
    Logger.log('CLIENT_ID and CLIENT_SECRET are not set yet.');
    return;
  }
  setPKCEChallengeVerifier();
  const service = getService();
  if (service.hasAccess()) {
    Logger.log('Already authorized.');
  } else {
    const authorizationUrl = service.getAuthorizationUrl();
    Logger.log('Authorize this app by visiting this URL: %s', authorizationUrl);
  }
}

function setPKCEChallengeVerifier() {
  const sp = PropertiesService.getScriptProperties();
  if (!sp.getProperty('CODE_VERIFIER')) {
    let verifier = '';
    const possible = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-._~';
    for (let i = 0; i < 128; i++) {
      verifier += possible.charAt(Math.floor(Math.random() * possible.length));
    }

    const sha256Hash = Utilities.computeDigest(Utilities.DigestAlgorithm.SHA_256, verifier);
    const challenge = Utilities.base64Encode(sha256Hash)
      .replace(/\+/g, '-')
      .replace(/\//g, '_')
      .replace(/=+$/, '');

    sp.setProperty('CODE_VERIFIER', verifier);
    sp.setProperty('CODE_CHALLENGE', challenge);
  }
}

function authCallback(request) {
  const service = getService();
  const isAuthorized = service.handleCallback(request);
  if (isAuthorized) {
    return HtmlService.createHtmlOutput('Success! You can close this tab.');
  } else {
    return HtmlService.createHtmlOutput('Denied. You can close this tab.');
  }
}

function resetService() {
  getService().reset();
  const sp = PropertiesService.getScriptProperties();
  sp.deleteProperty('CODE_VERIFIER');
  sp.deleteProperty('CODE_CHALLENGE');
  Logger.log('Access token and PKCE properties deleted. Rerun initialSetUp().');
}

function getService() {
  const sp = PropertiesService.getScriptProperties();
  const clid = sp.getProperty('CLIENT_ID');
  const clsec = sp.getProperty('CLIENT_SECRET');
  const verifier = sp.getProperty('CODE_VERIFIER');
  const challenge = sp.getProperty('CODE_CHALLENGE');

  return OAuth2.createService('twitter')
    .setAuthorizationBaseUrl('https://twitter.com/i/oauth2/authorize')
    .setTokenUrl('https://api.twitter.com/2/oauth2/token?code_verifier=' + verifier)
    .setClientId(clid)
    .setClientSecret(clsec)
    .setCallbackFunction('authCallback')
    .setPropertyStore(sp)
    .setScope('users.read tweet.read tweet.write offline.access')
    .setParam('response_type', 'code')
    .setParam('code_challenge_method', 'S256')
    .setParam('code_challenge', challenge)
    .setTokenHeaders({
      'Authorization': 'Basic ' + Utilities.base64Encode(clid + ':' + clsec),
      'Content-Type': 'application/x-www-form-urlencoded'
    })
}

function sendTweet(tweetText, targetID) {
  const payload = {
    text: tweetText,
    ...(targetID && {
      reply: {'in_reply_to_tweet_id': targetID}
    })
  };

  const service = getService();
  if (service.hasAccess()) {
    const url = 'https://api.twitter.com/2/tweets';
    const response = UrlFetchApp.fetch(url, {
      method: 'POST',
      contentType: 'application/json',
      headers: {
        Authorization: 'Bearer ' + service.getAccessToken()
      },  
      muteHttpExceptions: true,
      payload: JSON.stringify(payload)
    });
    const result = JSON.parse(response.getContentText());
    Logger.log(JSON.stringify(result, null, 2));
    return result.data.id;
  } else {
    Logger.log('No access authorization. Rerun initialSetUp().');
  }
}

function sendRandomTweet() {
  // trigger fires every 30min (48/day)
  if (!drawLots(36)) return;

  // [A:text, B:count, C:number, D:len(text), E:url(optional)]
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const range = sheet.getRange('A1:E');

  // generate random index [0, (LastRow - 1)]
  const i = Math.floor(Math.random() * sheet.getLastRow());

  // get (i + 1)th row in range
  const selectedRow = range.getValues()[i];
  const nextCount = selectedRow[1] + 1;
  sheet.getRange('B' + (i + 1)).setValue(nextCount);

  const targetID = sendTweet(selectedRow[0], null);
  if (selectedRow[4]) {
    Utilities.sleep(10000);
    sendTweet(selectedRow[4], targetID);
  }
}

function sendRareTweet() {
  // trigger fires every 30min (48/day)
  if (!drawLots(48 * 5)) return;

  // [A:text, B:count, C:number, D:len(text), E:url(optional)]
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const range = sheet.getRange('A1:E');

  // column 2 is count
  range.sort({column: 2, ascending: true});

  const selectedRow = range.getValues()[0];
  const nextCount = selectedRow[1] + 1;
  sheet.getRange('B1').setValue(nextCount);

  // cloumn 3 is serial number, restore normal order
  range.sort({column: 3, ascending: true});
  
  const targetID = sendTweet(selectedRow[0], null);
  if (selectedRow[4]) {
    Utilities.sleep(10000);
    sendTweet(selectedRow[4], targetID);
  }
}

function drawLots(num) {
  // true if 0, false otherwise, p=1/num
  return (Math.floor(Math.random() * num)) == 0;
}

function sendNewYearTweet() {
  const today = new Date();
  if (today.getMonth() != 0) return;

  // Happy New Year!
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const range = sheet.getRange('A1:D');

  const numlist = [29, 602];
  for (const num of numlist) {
    // num starts from 1, arrayindex from 0
    const selectedRow = range.getValues()[num - 1];
    const nextCount = selectedRow[1] + 1;
    sheet.getRange('B' + num).setValue(nextCount);

    sendTweet(selectedRow[0], null);
    if (num == numlist.at(-1)) break;
    Utilities.sleep(10000);
  }
}
```
