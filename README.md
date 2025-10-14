```
import boto3
from pydantic import BaseModel

# --- 提供されたモデル定義 ---
class Temp1Model(BaseModel):
    key1: str

class Temp2Model(BaseModel):
    key2: str

class Temp3Model(BaseModel):
    key3: str

class tempBase(BaseModel):
    Temp1: Temp1Model
    Temp2: Temp2Model
    Temp3: Temp3Model

# --- 提供されたデータオブジェクトの作成（仮） ---
# 実際の実装では、これらの値は動的に設定されます
temp1val = Temp1Model(key1="value1_updated")
temp2val = Temp2Model(key2="value2_updated")
temp3val = Temp3Model(key3="value3_updated")

update_items = tempBase(
    Temp1=temp1val,
    Temp2=temp2val,
    Temp3=temp3val,
)

# --- ここからが更新処理のメインロジック ---

# 1. boto3リソースを初期化
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('YourTableName') # << 実際のテーブル名に置き換えてください

# 2. 更新対象のアイテムのキーを定義
# このキーを持つアイテムが更新されます
primary_key = {'YourPrimaryKey': 'some-id-123'} # << 実際のプライマリーキーに置き換えてください

# 3. Pydanticオブジェクトを辞書に変換
# Pydantic V2では .model_dump() を使用します
update_dict = update_items.model_dump()
# Pydantic V1の場合は update_dict = update_items.dict()

# 4. update_item用のパラメータを動的に構築
update_expression_parts = []
expression_attribute_names = {}
expression_attribute_values = {}

for key, value in update_dict.items():
    name_placeholder = f"#{key}"  # 例: #Temp1
    value_placeholder = f":{key}" # 例: :Temp1

    update_expression_parts.append(f"{name_placeholder} = {value_placeholder}")
    expression_attribute_names[name_placeholder] = key
    
    # ネストされたオブジェクト(辞書)もそのまま値として渡せます
    expression_attribute_values[value_placeholder] = value

# "SET #Temp1 = :Temp1, #Temp2 = :Temp2, ..." という文字列を生成
update_expression = "SET " + ", ".join(update_expression_parts)

# 5. DynamoDBのアイテムを更新
try:
    print("🚀 DynamoDBのアイテムを更新します...")
    print(f"Key: {primary_key}")
    print(f"UpdateExpression: {update_expression}")
    # print(f"ExpressionAttributeNames: {expression_attribute_names}")
    # print(f"ExpressionAttributeValues: {expression_attribute_values}")

    response = table.update_item(
        Key=primary_key,
        UpdateExpression=update_expression,
        ExpressionAttributeNames=expression_attribute_names,
        ExpressionAttributeValues=expression_attribute_values,
        ReturnValues="UPDATED_NEW"  # 更新後の全属性値を返す
    )
    
    print("\n✅ 更新が成功しました。")
    print("更新後のデータ:", response.get('Attributes'))

except Exception as e:
    print(f"\n❌ 更新中にエラーが発生しました: {e}")
```
