```
def delete_all_items_for_partition_key(table_name, pk_name, pk_value, sk_name):
    """
    指定されたパーティションキーを持つすべてのDynamoDBアイテムを削除します。

    Args:
        table_name (str): 対象のテーブル名
        pk_name (str): パーティションキーの属性名
        pk_value (any): 削除対象のパーティションキーの値
        sk_name (str): ソートキーの属性名
    """
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table(table_name)
    
    # --- 1. 削除対象のアイテムをQueryで検索 ---
    print(f"Searching for items in '{table_name}' with {pk_name} = '{pk_value}'...")
    
    # キーを格納するリスト
    keys_to_delete = []
    
    # ページネーションを考慮して全アイテムを検索
    query_params = {
        'KeyConditionExpression': Key(pk_name).eq(pk_value)
    }
    
    while True:
        response = table.query(**query_params)
        items = response.get('Items', [])
        
        if not items:
            break # ループの初回でここに来たら、対象アイテムは0件

        for item in items:
            # batch_write_itemで削除するために、完全なキー（PKとSK）を収集
            keys_to_delete.append({
                pk_name: item[pk_name],
                sk_name: item[sk_name]
            })
            
        # 応答に 'LastEvaluatedKey' がなければ、それが最後のページ
        if 'LastEvaluatedKey' not in response:
            break
        
        # 次のページの開始キーを設定
        query_params['ExclusiveStartKey'] = response['LastEvaluatedKey']

    if not keys_to_delete:
        print("No items found to delete.")
        return

    # --- 2. batch_write_itemで一括削除 ---
    print(f"Found {len(keys_to_delete)} items. Starting batch deletion...")
    
    # batch_writerを使うと、25件ごとの分割やリトライ処理を自動的にハンドリングしてくれる
    with table.batch_writer() as batch:
        for key in keys_to_delete:
            batch.delete_item(Key=key)
            
    print("Deletion completed successfully.")
```
