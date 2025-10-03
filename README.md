はい、承知いたしました。
Pythonを使ったAWS LambdaでDynamoDBの項目を**登録・更新・取得**する3つの処理について、それぞれのサンプルコードを解説します。

ここでは、API GatewayとLambdaを連携させて、HTTPリクエストに応じて処理を振り分ける構成を想定しています。

-----

### 準備

#### 1\. DynamoDBテーブル

まず、DynamoDBにテーブルを作成しておきます。ここでは例として、以下のようなシンプルなテーブルを考えます。

  * **テーブル名**: `users`
  * **パーティションキー**: `userId` (文字列)

#### 2\. IAMロール

Lambda関数にアタッチするIAMロールには、DynamoDBへのアクセス許可が必要です。最低限、以下のポリシーをアタッチしてください。

  * `AmazonDynamoDBFullAccess` (テスト用) または、特定のテーブルに限定したより厳格なポリシー。

-----

### 1\. 登録 (Create)

新しい項目をDynamoDBに登録するLambda関数です。HTTPの`POST`メソッドで呼び出されることを想定しています。

**`create_item.py`**

```python
import json
import boto3
import os
import uuid

# DynamoDBリソースを初期化
dynamodb = boto3.resource('dynamodb')
# 環境変数からテーブル名を取得
table_name = os.environ.get('DYNAMODB_TABLE', 'users')
table = dynamodb.Table(table_name)

def lambda_handler(event, context):
    """
    HTTP POSTリクエストのボディに含まれるデータを使ってDynamoDBに新しい項目を登録する
    """
    try:
        # リクエストボディをJSONとしてパース
        body = json.loads(event.get('body', '{}'))

        # 必須項目の存在チェック
        if 'name' not in body or 'email' not in body:
            return {
                'statusCode': 400,
                'body': json.dumps({'error': 'name and email are required'})
            }

        # 新しい項目データを作成
        # userIdは自動で生成する
        item = {
            'userId': str(uuid.uuid4()),
            'name': body['name'],
            'email': body['email']
        }

        # DynamoDBに項目を登録 (put_item)
        table.put_item(Item=item)

        # 成功レスポンスを返す
        return {
            'statusCode': 201, # 201 Created
            'headers': {
                'Content-Type': 'application/json'
            },
            'body': json.dumps(item)
        }

    except Exception as e:
        print(f"Error: {e}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }
```

**このコードのポイント**

  * **`uuid.uuid4()`**: 新しいユーザーのために一意のIDを自動生成しています。
  * **`table.put_item(Item=item)`**: DynamoDBに新しい項目を追加する中心的な処理です。`Item`引数に辞書形式でデータを渡します。

-----

### 2\. 更新 (Update)

既存の項目を更新するLambda関数です。HTTPの`PUT`または`PATCH`メソッドで呼び出されることを想定しています。

**`update_item.py`**

```python
import json
import boto3
import os

dynamodb = boto3.resource('dynamodb')
table_name = os.environ.get('DYNAMODB_TABLE', 'users')
table = dynamodb.Table(table_name)

def lambda_handler(event, context):
    """
    指定されたuserIdの項目を更新する
    """
    try:
        # URLのパスパラメータからuserIdを取得
        path_parameters = event.get('pathParameters', {})
        user_id = path_parameters.get('userId')

        if not user_id:
            return {
                'statusCode': 400,
                'body': json.dumps({'error': 'userId is required in path'})
            }

        # リクエストボディから更新内容を取得
        body = json.loads(event.get('body', '{}'))

        # 更新式と属性値を動的に構築
        update_expression = "SET "
        expression_attribute_values = {}
        expression_attribute_names = {} # 予約語を避けるために使用
        
        # 更新可能なフィールドをループ
        for key, value in body.items():
            # キーが予約語でないことを確認し、安全なプレースホルダーを作成
            placeholder_key = f"#{key}"
            placeholder_value = f":{key}"
            update_expression += f"{placeholder_key} = {placeholder_value}, "
            expression_attribute_names[placeholder_key] = key
            expression_attribute_values[placeholder_value] = value

        # 最後のカンマとスペースを削除
        update_expression = update_expression.rstrip(', ')

        if not expression_attribute_values:
             return {
                'statusCode': 400,
                'body': json.dumps({'error': 'No update fields provided'})
            }

        # DynamoDBの項目を更新 (update_item)
        response = table.update_item(
            Key={'userId': user_id},
            UpdateExpression=update_expression,
            ExpressionAttributeValues=expression_attribute_values,
            ExpressionAttributeNames=expression_attribute_names,
            ReturnValues="UPDATED_NEW"  # 更新後の項目を返す
        )

        return {
            'statusCode': 200,
            'headers': {
                'Content-Type': 'application/json'
            },
            'body': json.dumps(response['Attributes'])
        }

    except Exception as e:
        print(f"Error: {e}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }
```

**このコードのポイント**

  * **`pathParameters`**: API Gatewayで `/users/{userId}` のようにパスを設定した場合、`userId`をここから取得します。
  * **`UpdateExpression`**: `"SET #name = :name, #email = :email"` のような更新クエリを動的に作成します。
  * **`ExpressionAttributeNames`**: `name`がDynamoDBの予約語である可能性があるため、`#name`のようなプレースホルダーで安全に扱います。
  * **`table.update_item(...)`**: 項目を更新するメソッドです。`Key`で対象を指定し、`UpdateExpression`で更新内容を指示します。

-----

### 3\. 取得 (Get)

特定の項目を取得するLambda関数です。HTTPの`GET`メソッドで呼び出されることを想定しています。

**`get_item.py`**

```python
import json
import boto3
import os

dynamodb = boto3.resource('dynamodb')
table_name = os.environ.get('DYNAMODB_TABLE', 'users')
table = dynamodb.Table(table_name)

def lambda_handler(event, context):
    """
    指定されたuserIdの項目を取得する
    """
    try:
        # クエリパラメータまたはパスパラメータからuserIdを取得
        user_id = event.get('queryStringParameters', {}).get('userId')
        if not user_id:
             user_id = event.get('pathParameters', {}).get('userId')

        if not user_id:
            return {
                'statusCode': 400,
                'body': json.dumps({'error': 'userId is required'})
            }

        # DynamoDBから項目を取得 (get_item)
        response = table.get_item(
            Key={
                'userId': user_id
            }
        )

        # 項目が存在するかチェック
        item = response.get('Item')
        if item:
            return {
                'statusCode': 200,
                'headers': {
                    'Content-Type': 'application/json'
                },
                'body': json.dumps(item)
            }
        else:
            return {
                'statusCode': 404, # 404 Not Found
                'body': json.dumps({'error': 'User not found'})
            }

    except Exception as e:
        print(f"Error: {e}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }
```

**このコードのポイント**

  * **`queryStringParameters`**: API Gatewayで `/users?userId=xxx` のようなリクエストを想定しています。
  * **`table.get_item(Key=key)`**: 指定されたキー（この場合は`userId`）に一致する単一の項目を取得します。
  * **`response.get('Item')`**: 取得結果は`'Item'`というキーの中に格納されています。存在しない場合は`None`が返るため、404エラーのハンドリングに使えます。

これらのコードをベースに、ご自身の要件に合わせてエラーハンドリングや入力バリデーションを強化してご活用ください。
