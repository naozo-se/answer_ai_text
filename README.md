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

