```
import boto3
from botocore.exceptions import ClientError
from pydantic import ValidationError

# --- 上で定義したPydanticモデル ---
from pydantic import BaseModel, Field
from typing import Optional

class User(BaseModel):
    user_id: str = Field(..., description="ユーザーID")
    username: str = Field(..., description="ユーザー名")
    email: str = Field(..., description="メールアドレス")
    age: Optional[int] = Field(None, description="年齢")
# ---------------------------------


def get_user_from_dynamodb(table_name: str, user_id: str) -> Optional[User]:
    """
    DynamoDBテーブルから指定されたキーのアイテムを取得し、Pydanticモデルで検証する

    Args:
        table_name (str): DynamoDBのテーブル名
        user_id (str): 取得したいアイテムのパーティションキー

    Returns:
        Optional[User]: 見つかった場合はUserオブジェクト、見つからない場合や
                        バリデーションに失敗した場合はNone
    """
    # DynamoDBリソースの初期化
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table(table_name)

    try:
        # get_itemメソッドでデータを取得
        response = table.get_item(
            Key={
                'user_id': user_id
            }
        )
        
        item = response.get('Item')

        if not item:
            print(f"ユーザーが見つかりませんでした。user_id: {user_id}")
            return None

        # 取得したデータをPydanticモデルでバリデーション
        try:
            user = User(**item)
            return user
        except ValidationError as e:
            print(f"データのバリデーションに失敗しました: {e}")
            return None

    except ClientError as e:
        print(f"データの取得中にエラーが発生しました: {e.response['Error']['Message']}")
        return None
    except Exception as e:
        print(f"予期せぬエラーが発生しました: {e}")
        return None


# --- 使用例 ---
if __name__ == '__main__':
    TABLE_NAME = 'users'  # ご自身のテーブル名に変更してください
    TARGET_USER_ID = 'user123'   # 取得したいユーザーID

    # AWS認証情報が設定されている環境で実行してください
    # (例: IAMロール、~/.aws/credentials、環境変数など)
    user_data = get_user_from_dynamodb(TABLE_NAME, TARGET_USER_ID)

    if user_data:
        print("取得したユーザー情報:")
        print(f"  ID: {user_data.user_id}")
        print(f"  Name: {user_data.username}")
        print(f"  Email: {user_data.email}")
        print(f"  Age: {user_data.age if user_data.age is not None else 'N/A'}")
```
