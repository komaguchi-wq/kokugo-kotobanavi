# Google Sheets バックアップ・復元 設定手順

## 1. Google Sheets を作成

新しいスプレッドシートを作成する（名前は「国語 言葉ナビ」など自由）。

## 2. Apps Script を設定

スプレッドシートを開き、メニューから「拡張機能 → Apps Script」を選択。
以下のコードを貼り付けて保存する。

```javascript
function doPost(e) {
  try {
    const data = JSON.parse(e.postData.contents);
    const ss = SpreadsheetApp.getActiveSpreadsheet();

    // ユーザーごとのシートを取得または作成
    let sheet = ss.getSheetByName(data.user);
    if (!sheet) {
      sheet = ss.insertSheet(data.user);
      sheet.appendRow(["timestamp", "unit", "question", "attempts", "correct", "accuracy"]);
    }

    // 既存データをクリアして最新を書き込み
    if (sheet.getLastRow() > 1) {
      sheet.getRange(2, 1, sheet.getLastRow() - 1, 6).clear();
    }

    let row = 2;
    for (const unitId in data.data) {
      for (const key in data.data[unitId]) {
        const record = data.data[unitId][key];
        const accuracy = record.attempts > 0
          ? Math.round((record.correct / record.attempts) * 100) + "%"
          : "---";
        sheet.getRange(row, 1, 1, 6).setValues([[
          data.timestamp,
          unitId,
          key,
          record.attempts,
          record.correct,
          accuracy
        ]]);
        row++;
      }
    }

    return ContentService.createTextOutput(
      JSON.stringify({ status: "ok" })
    ).setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService.createTextOutput(
      JSON.stringify({ status: "error", message: err.toString() })
    ).setMimeType(ContentService.MimeType.JSON);
  }
}

function doGet(e) {
  try {
    const user = e.parameter.user;
    if (!user) {
      return ContentService.createTextOutput(
        JSON.stringify({ status: "error", message: "user parameter required" })
      ).setMimeType(ContentService.MimeType.JSON);
    }

    const ss = SpreadsheetApp.getActiveSpreadsheet();
    const sheet = ss.getSheetByName(user);

    if (!sheet || sheet.getLastRow() <= 1) {
      return ContentService.createTextOutput(
        JSON.stringify({ status: "ok", data: {} })
      ).setMimeType(ContentService.MimeType.JSON);
    }

    const rows = sheet.getRange(2, 1, sheet.getLastRow() - 1, 6).getValues();
    const tracking = {};

    for (const row of rows) {
      const unitId = row[1];
      const question = row[2];
      const attempts = row[3];
      const correct = row[4];

      if (!unitId || !question) continue;
      if (!tracking[unitId]) tracking[unitId] = {};
      tracking[unitId][question] = { attempts: attempts, correct: correct };
    }

    return ContentService.createTextOutput(
      JSON.stringify({ status: "ok", data: tracking })
    ).setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService.createTextOutput(
      JSON.stringify({ status: "error", message: err.toString() })
    ).setMimeType(ContentService.MimeType.JSON);
  }
}
```

## 3. ウェブアプリとしてデプロイ

1. 「デプロイ → 新しいデプロイ」
2. 種類: 「ウェブアプリ」
3. 実行するユーザー: 「自分」
4. アクセスできるユーザー: 「全員」
5. デプロイして URL をコピー

## 4. アプリに URL を設定

アプリの単元一覧画面の右上にある ⚙ ボタンから設定画面を開き、URLを貼り付けて「保存」を押す。

## 5. データの復元

別の端末でアプリを開いたら：
1. ⚙ ボタンで設定画面を開く
2. 同じURLを貼り付けて「保存」
3. 「スプレッドシートから復元」ボタンを押す

## データ構造

スプレッドシートにはユーザーごとのシートが作られ、以下の列で記録される:

| timestamp | unit | question | attempts | correct | accuracy |
|-----------|------|----------|----------|---------|----------|
| 2026-03-09T... | 1-2 | 5-0 | 3 | 2 | 67% |
| 2026-03-09T... | 1-2 | 5-1 | 2 | 2 | 100% |
