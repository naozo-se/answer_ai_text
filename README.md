```
# 1. 検索したいキーと値を辞書で定義
conditions = {
    'customer_id': 'cust123',
    'status': 'shipped'
}

# 2. forループで条件式を組み立てる
key_condition_expr = None
for k, v in conditions.items():
    current_condition = Key(k).eq(v)
    if key_condition_expr is None:
        # 最初の条件
        key_condition_expr = current_condition
    else:
        # 2つ目以降の条件を '&' で結合
        key_condition_expr = key_condition_expr & current_condition
```
