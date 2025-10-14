```
# Userクラスのインスタンスを update_items とする
update_dict = update_items.model_dump()

update_expression_parts = []
expression_attribute_names = {}
expression_attribute_values = {}

for key, value in update_dict.items():
    name_placeholder = f"#{key}"
    value_placeholder = f":{key}"

    # --- ここからが変更点 ---
    # モデルの定義から、現在のキーに対応するフィールド情報を取得
    field_info = User.model_fields[key]
    
    # フィールド情報に db_update_strategy='if_not_exists' が設定されているかチェック
    # getattr()を使うと、属性が存在しなくてもエラーにならないので安全です
    if getattr(field_info, 'db_update_strategy', None) == 'if_not_exists':
        # 設定があれば、if_not_existsを使った文字列を生成
        update_expression_parts.append(
            f"{name_placeholder} = if_not_exists({name_placeholder}, {value_placeholder})"
        )
    else:
        # 設定がなければ、通常の上書き文字列を生成
        update_expression_parts.append(f"{name_placeholder} = {value_placeholder}")
    # --- 変更点はここまで ---
    
    expression_attribute_names[name_placeholder] = key
    expression_attribute_values[value_placeholder] = value

update_expression = "SET " + ", ".join(update_expression_parts)
```
