```python
import boto3
from boto3.dynamodb.conditions import Key
import os

# ==========================================
# 設定項目: ここを書き換えてください
# ==========================================
TABLE_NAME = "TargetTableName"
REGION_NAME = os.environ.get('AWS_DEFAULT_REGION', 'ap-northeast-1')

# 条件1: Partition Key (PK) の設定
PK_NAME = "UserId"          # パーティションキーのカラム名
PK_VALUE = "user_001"       # 削除対象の値 (完全一致)

# 条件2: Sort Key (SK) の設定
SK_NAME = "Timestamp"       # ソートキーのカラム名
SK_PREFIX = "2024-01"       # この文字で始まるものを削除 (前方一致)
# ==========================================

def delete_specific_items(table_name, region):
    dynamodb = boto3.resource('dynamodb', region_name=region)
    table = dynamodb.Table(table_name)
    
    print(f"Target Table: {table_name}")
    print(f"Condition: {PK_NAME} = '{PK_VALUE}' AND {SK_NAME} starts with '{SK_PREFIX}'")
    
    # 削除用バッチライター
    with table.batch_writer() as batch:
        # Queryのためのパラメータ
        query_kwargs = {
            # Keyオブジェクトを使うと直感的にAND条件が書けます
            'KeyConditionExpression': Key(PK_NAME).eq(PK_VALUE) & Key(SK_NAME).begins_with(SK_PREFIX),
            
            # 削除に必要なのはキーだけなので、転送量を減らすためにキーのみ取得
            # (ProjectionExpressionは必須ではありませんが推奨です)
            'ProjectionExpression': f"#{PK_NAME}, #{SK_NAME}",
            'ExpressionAttributeNames': {
                f"#{PK_NAME}": PK_NAME,
                f"#{SK_NAME}": SK_NAME
            }
        }
        
        done = False
        start_key = None
        count = 0
        
        while not done:
            if start_key:
                query_kwargs['ExclusiveStartKey'] = start_key
            
            # scanではなくqueryを実行
            response = table.query(**query_kwargs)
            items = response.get('Items', [])
            
            for item in items:
                # バッチ削除リストに追加
                batch.delete_item(Key={
                    PK_NAME: item[PK_NAME],
                    SK_NAME: item[SK_NAME]
                })
                count += 1
            
            start_key = response.get('LastEvaluatedKey', None)
            done = start_key is None
            print(f"\rQueried and queued {count} items...", end="")
            
    print(f"\nCompleted. Deleted {count} items.")

if __name__ == "__main__":
    delete_specific_items(TABLE_NAME, REGION_NAME)
```
