```
import boto3
import os

# ==========================================
# 設定: 対象のテーブル名
TABLE_NAME = "TargetTableName"

# 必要であればリージョンを指定（CloudShellと同じならそのままでOK）
REGION_NAME = os.environ.get('AWS_DEFAULT_REGION', 'ap-northeast-1')
# ==========================================

def truncate_dynamodb_table(table_name, region):
    dynamodb = boto3.resource('dynamodb', region_name=region)
    table = dynamodb.Table(table_name)
    
    # テーブルのキー設定（パーティションキー、ソートキー）を自動取得
    # これにより、キー名が "id" 以外でも、複合キーでも動作します
    key_names = [key['AttributeName'] for key in table.key_schema]
    projection_expression = ", ".join([f"#{k}" for k in key_names])
    expression_attribute_names = {f"#{k}": k for k in key_names}
    
    print(f"Target Table: {table_name} ({region})")
    print(f"Detected Keys: {key_names}")
    print("Starting truncation...")

    with table.batch_writer() as batch:
        scan_kwargs = {
            'ProjectionExpression': projection_expression,
            'ExpressionAttributeNames': expression_attribute_names
        }
        
        done = False
        start_key = None
        count = 0
        
        while not done:
            if start_key:
                scan_kwargs['ExclusiveStartKey'] = start_key
            
            response = table.scan(**scan_kwargs)
            items = response.get('Items', [])
            
            for item in items:
                # 取得したキー情報を使って削除キーを構築
                key_dict = {k: item[k] for k in key_names}
                batch.delete_item(Key=key_dict)
                count += 1
            
            start_key = response.get('LastEvaluatedKey', None)
            done = start_key is None
            print(f"\rScanned and queued {count} items...", end="")
            
    print(f"\nCompleted. Deleted {count} items.")

if __name__ == "__main__":
    truncate_dynamodb_table(TABLE_NAME, REGION_NAME)
```

```
(Get-Content "ファイルのパス" -Raw).Replace("`r`n","`n") | Set-Content "ファイルのパス" -NoNewline
```


```
# 1. 対象のテーブル名を設定（ここを書き換えてください）
export TABLE_NAME="TargetTableName"

# 2. scanして整形し、ファイルに出力
aws dynamodb scan --table-name $TABLE_NAME --output json | \
jq --arg table "$TABLE_NAME" \
'{($table): [.Items[] | {PutRequest: {Item: .}}]}' > backup.json
```


```
find . -maxdepth 1 -name "temp.*.txt" | sort -V | tail -n 1
```

```
import boto3
import time
import json

# ==========================================
# 設定エリア
# ==========================================
TABLE_NAME = 'YourTableName'   # DynamoDBのテーブル名
REGION = 'ap-northeast-1'      # リージョン
TEMPLATE_FILE = 'format.json'  # テンプレートファイル名

# ループ設定
START_INDEX = 0        # 開始番号
END_INDEX   = 10000    # 終了番号 (この番号の手前まで実行)
# ==========================================

def generate_data_from_template():
    """
    テンプレートファイルを読み込み、設定された範囲でデータを生成する
    """
    try:
        with open(TEMPLATE_FILE, 'r') as f:
            template_str = f.read()
    except FileNotFoundError:
        # print(f"エラー: {TEMPLATE_FILE} が見つかりません。")
        return

    # 設定された範囲でループ
    for i in range(START_INDEX, END_INDEX):
        
        current_time = int(time.time())
        
        # 文字列置換
        record_str = template_str.replace('(i)', str(i))
        record_str = record_str.replace('(timestamp)', str(current_time))
        
        try:
            item = json.loads(record_str)
            yield item
        except json.JSONDecodeError as e:
            # print(f"JSON変換エラー (index: {i}): {e}")
            break

def main():
    dynamodb = boto3.resource('dynamodb', region_name=REGION)
    table = dynamodb.Table(TABLE_NAME)

    # print(f"処理開始: {START_INDEX} から {END_INDEX - 1} まで投入します")
    # start_time = time.time()

    # count = 0
    with table.batch_writer() as batch:
        for item in generate_data_from_template():
            batch.put_item(Item=item)
            # count += 1
            
            # 進捗表示もコメントアウト
            # if count % 1000 == 0:
            #    print(f"{count} 件処理中...")

    # end_time = time.time()
    # duration = end_time - start_time
    # print("-" * 30)
    # print(f"完了しました。")
    # print(f"投入件数: {count} 件")
    # print(f"所要時間: {duration:.2f} 秒")

if __name__ == '__main__':
    main()
```

```
{
    "partition_key": "temp_(i)",
    "sort_key": "sort_(i)",
    "timestamp": (timestamp),
    "constant": "aaa"
}
```

