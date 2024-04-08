---
title: "GAS + API Gateway + Lambda を使って DB からデータを取得し Spreadsheet に貼り付ける"
emoji: "⛳"
type: "tech"
topics:
  - "aws"
  - "lambda"
  - "gas"
  - "googlespreadsheet"
  - "chalice"
published: true
published_at: "2024-03-10 16:26"
---

## 背景・目的
趣味プロジェクトで、最低限の分析やモニタリングのため、DB (Postgres, on Render) のデータを自分が好きなタイミングにクエリしてSpreadsheetに貼り付ける作業を自動化したい。

## 実装方法
Google App Script (GAS) + API Gateway + Lambda を使って実装した。

当初、下記の記事を参考し、 GAS 内で JDBC を使用して直接 DB にアクセスしようと考えたが、GAS で使う JDBC サービスが Postgres にサポートしていなかったため断念。
https://qiita.com/kazu_wsx/items/b01b810cc04390907603
https://developers.google.com/apps-script/guides/jdbc

そのため、一度 API Gateway + Lambda で API を作り、GAS から API を呼び出す形にした。

## Chalice で API Gateway - Lambda デプロイ
API Gateway + Lambda の構築には Chalice を使用した。Chalice を使うと、コマンド操作だけで IAM ロール等を含めた権限設定、API Gateway と Lambda のセットアップ、API Gateway のエンドポイント発行までを自動で行ってくれるため、非常に便利。

Chalice の参考記事
https://www.cloudbuilders.jp/articles/2265/

Chalice の基本的なコマンドをここにメモっておく
- `pip install --upgrade chalice`：Chalice のアップグレード
- `chalice new-project project-name`：新しいプロジェクトの作成
- `chalice local`：ローカル環境でのテスト
- `chalice deploy`：AWS へのデプロイ
- `chalice url`：デプロイされた API の URL 取得

※ 以前は Chalice の古いバージョンをインストールしてあり、アップグレードしてない状態で使ってたら Python のバージョンの問題で引っ掛かり時間がかかったため最初にアップグレードを忘れないように。最近結構更新されてるっぽいし。

Lambda 側のコード：
```python
from chalice import Chalice
import json
import psycopg2

app = Chalice(app_name='your_project_name')

@app.route('/')
def index():
    host = "{dbhost}"
    dbname = "{dbname}"
    user = "{user}"
    password = "{pw}"
    port = "5432"

    conn = psycopg2.connect(
        host=host,
        dbname=dbname,
        user=user,
        password=password,
        port=port
    )

    cursor = conn.cursor()

    sql = """
    {SQL はここに書く}
    """
    cursor.execute(sql)

    rows = cursor.fetchall()
    result = []
    for row in rows:
        result.append({
            "id": row[0],
            "name": row[1],
            "search_keyword": row[2],
            "created_at": row[3].strftime("%Y-%m-%d %H:%M:%S")
        })

    cursor.close()
    conn.close()

    return {
        'statusCode': 200, 
        'body': json.dumps(result)
    }
```

なお、Lambda で psycopg2 を使用する際は、requirements.txt で psycopg2-binary を指定しないとエラーがでるので要注意。

## GAS の実装
GAS 側では以下の処理を行う。
1. `chalice url` で出力された API Gateway のエンドポイントを呼び出す
2. 受け取った JSON を整形してスプレッドシートに貼り付ける  
3. ボタンを押すとスクリプトが実行され、データが更新されるようにする

```javascript
function insertDataIntoSpreadsheet() {
  var spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
  var sheet = spreadsheet.getSheetByName("search_history");
  if (!sheet) {
    Logger.log("Sheet 'search_history' not found.");
    return;
  }
  
  var apiUrl = '{Lambda関数を呼び出すURLをここ}';
  var response = UrlFetchApp.fetch(apiUrl);
  var responseData = JSON.parse(response.getContentText());
  var data = responseData.body;
  
  if (!data) {
    Logger.log("No data received");
    return;
  }
  
  var rows = JSON.parse(data);
  var values = [];
  
  values.push(["ID", "Name", "Search Keyword", "Created At"]);
  
  rows.forEach(function(row) {
    values.push([row.id, row.name, row.search_keyword, row.created_at]);
  });

  // 最初の5列を選択して既存の内容をクリアする
  var range = sheet.getRange(1, 1, sheet.getMaxRows(), 4);
  range.clearContent();

  var range = sheet.getRange(1, 1, values.length, 4); 
  range.setValues(values);
}
```

当初、GAS 側を毎時間自動更新することを検討したが、スプレッドシート側でコントロールできる方が良いと判断した。またスマホ側で更新操作ができるようにするため、今回は G2 セルにチェックボックスを挿入し、チェック時に更新されるようにした。 

下記の記事を参考
https://qiita.com/neras_1215/items/5dea01aecda9f93935bd

ただし、シンプルトリガーの OnEdit では外部 API の操作ができない。下記のエラーが出る。

```
Error	Exception: You do not have permission to call UrlFetchApp.fetch. Required permissions: https://www.googleapis.com/auth/script.external_request
    at insertDataIntoSpreadsheet(Code:11:30)
    at onEdit(Code:50:7)
```

そのため、カスタムトリガーを作る必要があった。ここでは `customOnEdit` を作成した。下記のコードを追加した上で、
スクリプトエディターのトリガーボタン（時計アイコン）から、トリガーを追加、 `customOnEdit` を選択し、`Select event source` は `From Spreadsheet`, `select event type` は `on edit` に設定することで、カスタムトリガーが設定される。
カスタムトリガーの場合は外部 API を呼び出せるようになる。

![](https://storage.googleapis.com/zenn-user-upload/5d8c72051165-20240310.png)

```javascript  
function customOnEdit(e) {
  var range = e.range;
  var sheet = range.getSheet();
  var sheetName = sheet.getName();
  
  if (sheetName === "search_history" && range.getA1Notation() === "G2") {
    var isChecked = range.getValue() === true;
    if (isChecked) {
      insertDataIntoSpreadsheet();
      range.setValue(false); // 実施完了したらチェックボックスのチェックを外す
    }
  }
}
```

## まとめ
今回は趣味プロジェクトということでセキュリティ関連の設定は最小限に留めたが、GAS + API Gateway + Lambda を使うことで、 DB から好きなタイミングにデータを取得し Spreadsheet に貼り付ける作業を自動化することができた。Chalice を使うことで Lambda の構築が非常に簡単になり、GAS から API を呼び出すことでスプレッドシート上での操作性も確保できた。