```
import boto3

# DynamoDBリソースを取得
dynamodb = boto3.resource('dynamodb')
# 'YourTableName', 'YourPKName', 'YourSKName' は実際の値に置き換えてください
table_name = 'YourTableName'
pk_name = 'YourPKName' # (例: 'UserID')
sk_name = 'YourSKName' # (例: 'Timestamp')
table = dynamodb.Table(table_name)

# PKをキーとし、SKが最大のアイテムを保持する辞書
max_sk_items = {}

# Scanオペレーションのパラメータ
scan_kwargs = {}

print(f"Scanning table '{table_name}' to find max SK for each PK...")

try:
    # ページネーション（データが1MBを超える場合）に対応
    done = False
    start_key = None
    total_scanned = 0
    
    while not done:
        if start_key:
            scan_kwargs['ExclusiveStartKey'] = start_key
        
        response = table.scan(**scan_kwargs)
        
        items = response.get('Items', [])
        total_scanned += len(items)
        
        for item in items:
            pk_val = item[pk_name]
            sk_val = item[sk_name]
            
            # 1. このPKがまだ辞書にない場合
            # 2. または、このPKの既存アイテムより、現在のアイテムのSKが大きい場合
            if pk_val not in max_sk_items or sk_val > max_sk_items[pk_val][sk_name]:
                # アイテムを更新（上書き）
                max_sk_items[pk_val] = item
                
        # 次のスキャン開始位置を取得
        start_key = response.get('LastEvaluatedKey', None)
        # LastEvaluatedKey がなければスキャン終了
        done = start_key is None

    # 辞書の値（Items）をリストに変換
    final_results = list(max_sk_items.values())

    print(f"\nScan complete. Scanned {total_scanned} total items.")
    print(f"Found {len(final_results)} unique PKs with their max SK item.")
    
    # # (オプション) 結果の表示
    # for item in final_results:
    #     print(item)

except Exception as e:
    print(f"Error scanning table: {e}")
    
```
