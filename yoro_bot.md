<img src="/images/yoro_bot_400x400.png" alt="yoro_bot icon image" width="100" height="100" />

### 更新履歴

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

- 原テキスト中で段落が替わっている箇所には、ツイートでは「　」（全角スペース）を挿入しています。
- 原テキスト中で少し離れた２つ以上の部分を１つのツイートにまとめる場合は、「／」（全角スラッシュ）でその分かれ目を示しています。
- 文脈がないと分かりづらい箇所には、「（※）」（全角アスタリスク）に続く丸括弧内で管理者による註釈を加えています。

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
|未|『身体観』|『日本人の身体観の歴史』|

### コード.gs

```javascript
const CLIENT_ID = '<Client ID>'
const CLIENT_SECRET = '<Client Secret>'

function getService() {
  pkceChallengeVerifier();
  const userProps = PropertiesService.getUserProperties();
  const scriptProps = PropertiesService.getScriptProperties();
  return OAuth2.createService('twitter')
    .setAuthorizationBaseUrl('https://twitter.com/i/oauth2/authorize')
    .setTokenUrl('https://api.twitter.com/2/oauth2/token?code_verifier=' + userProps.getProperty("code_verifier"))
    .setClientId(CLIENT_ID)
    .setClientSecret(CLIENT_SECRET)
    .setCallbackFunction('authCallback')
    .setPropertyStore(userProps)
    .setScope('users.read tweet.read tweet.write offline.access')
    .setParam('response_type', 'code')
    .setParam('code_challenge_method', 'S256')
    .setParam('code_challenge', userProps.getProperty("code_challenge"))
    .setTokenHeaders({
      'Authorization': 'Basic ' + Utilities.base64Encode(CLIENT_ID + ':' + CLIENT_SECRET),
      'Content-Type': 'application/x-www-form-urlencoded'
    })
}

function authCallback(request) {
  const service = getService();
  const authorized = service.handleCallback(request);
  if (authorized) {
    return HtmlService.createHtmlOutput('Success!');
  } else {
    return HtmlService.createHtmlOutput('Denied.');
  }
}

function pkceChallengeVerifier() {
  var userProps = PropertiesService.getUserProperties();
  if (!userProps.getProperty("code_verifier")) {
    var verifier = "";
    var possible = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-._~";

    for (var i = 0; i < 128; i++) {
      verifier += possible.charAt(Math.floor(Math.random() * possible.length));
    }

    var sha256Hash = Utilities.computeDigest(Utilities.DigestAlgorithm.SHA_256, verifier)

    var challenge = Utilities.base64Encode(sha256Hash)
      .replace(/\+/g, '-')
      .replace(/\//g, '_')
      .replace(/=+$/, '')
    userProps.setProperty("code_verifier", verifier)
    userProps.setProperty("code_challenge", challenge)
  }
}

function logRedirectUri() {
  var service = getService();
  Logger.log(service.getRedirectUri());
}

function sendTweet(tweetText) {
  var payload = { text: tweetText };
  //Logger.log(payload);

  var service = getService();
  if (service.hasAccess()) {
    var url = `https://api.twitter.com/2/tweets`;
    var response = UrlFetchApp.fetch(url, {
      method: 'POST',
      'contentType': 'application/json',
      headers: {
        Authorization: 'Bearer ' + service.getAccessToken()
      },
      muteHttpExceptions: true,
      payload: JSON.stringify(payload)
    });
    var result = JSON.parse(response.getContentText());
    Logger.log(JSON.stringify(result, null, 2));
  } else {
    var authorizationUrl = service.getAuthorizationUrl();
    Logger.log('Open the following URL and re-run the script: %s', authorizationUrl);
  }
}

function sendRandomTweet() {
  // trigger fires every 30min (48/day)
  if (!drawLots(48)) return;

  // [A:text, B:count, C:number, D:len(text)]
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const range = sheet.getRange("A1:D");

  // generate random index [0, (LastRow - 1)]
  const i = Math.floor(Math.random() * sheet.getLastRow());

  // get (i + 1)th row in range
  const selectedRow = range.getValues()[i];
  //Logger.log(selectedRow);
  const nextCount = selectedRow[1] + 1;
  sheet.getRange("B" + (i + 1)).setValue(nextCount);

  sendTweet(selectedRow[0]);
}

function sendRareTweet() {
  // trigger fires every 30min (48/day)
  if (!drawLots(48 * 5)) return;

  // [A:text, B:count, C:number, D:len(text)]
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const range = sheet.getRange("A1:D");

  // column 2 is count
  range.sort({column: 2, ascending: true});

  const selectedRow = range.getValues()[0];
  //Logger.log(selectedRow)
  const nextCount = selectedRow[1] + 1;
  sheet.getRange("B1").setValue(nextCount);

  // cloumn 3 is serial number, restore normal order
  range.sort({column: 3, ascending: true});
  
  sendTweet(selectedRow[0]);
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
  const range = sheet.getRange("A1:D");

  const numlist = [29, 602];
  for (const num of numlist) {
    // num starts from 1, arrayindex from 0
    const selectedRow = range.getValues()[num - 1];
    const nextCount = selectedRow[1] + 1;
    sheet.getRange("B" + num).setValue(nextCount);

    sendTweet(selectedRow[0]);
    if (num == numlist.slice(-1)[0]) break;
    Utilities.sleep(10000);
  }
}
```
