```python
import boto3
import os
from boto3.dynamodb.conditions import Attr

# ==========================================
# 設定
TABLE_NAME = "TargetTableName"

# 削除対象の「ソートキー」がこの文字で始まる場合のみ削除します
# 例: "test-" と設定すれば "test-001", "test-user" などが削除対象になります
TARGET_PREFIX = "test_" 

# 必要であればリージョンを指定
REGION_NAME = os.environ.get('AWS_DEFAULT_REGION', 'ap-northeast-1')
# ==========================================

def truncate_dynamodb_table_filtered(table_name, region, prefix):
    dynamodb = boto3.resource('dynamodb', region_name=region)
    table = dynamodb.Table(table_name)
    
    # キー情報の取得
    # KeyType='HASH' -> パーティションキー
    # KeyType='RANGE' -> ソートキー
    key_names = [key['AttributeName'] for key in table.key_schema]
    sort_key_name = next((key['AttributeName'] for key in table.key_schema if key['KeyType'] == 'RANGE'), None)

    # 削除に必要なキー情報のみを取得するための設定
    projection_expression = ", ".join([f"#{k}" for k in key_names])
    expression_attribute_names = {f"#{k}": k for k in key_names}
    
    print(f"Target Table: {table_name} ({region})")
    
    # フィルタ条件の作成
    scan_kwargs = {
        'ProjectionExpression': projection_expression,
        'ExpressionAttributeNames': expression_attribute_names
    }

    if prefix and sort_key_name:
        print(f"Filter Condition: SortKey '{sort_key_name}' begins with '{prefix}'")
        # ここでフィルタ条件を追加
        scan_kwargs['FilterExpression'] = Attr(sort_key_name).begins_with(prefix)
    elif prefix and not sort_key_name:
        print("Warning: Prefix was provided, but this table does not have a Sort Key. Aborting to be safe.")
        return
    else:
        print("No prefix provided. Warning: This will delete ALL data.")
        # 安全のため、prefix未指定時は確認を入れる等の対策をしても良いですが、今回はそのまま進めます
    
    print("Starting conditional deletion...")

    with table.batch_writer() as batch:
        done = False
        start_key = None
        count = 0
        
        while not done:
            if start_key:
                scan_kwargs['ExclusiveStartKey'] = start_key
            
            response = table.scan(**scan_kwargs)
            items = response.get('Items', [])
            
            for item in items:
                # 削除キーを構築
                key_dict = {k: item[k] for k in key_names}
                batch.delete_item(Key=key_dict)
                count += 1
            
            start_key = response.get('LastEvaluatedKey', None)
            done = start_key is None
            print(f"\rScanned and queued {count} items...", end="")
            
    print(f"\nCompleted. Deleted {count} items matching '{prefix}'.")

if __name__ == "__main__":
    truncate_dynamodb_table_filtered(TABLE_NAME, REGION_NAME, TARGET_PREFIX)
```
