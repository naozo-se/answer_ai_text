```
from pydantic import BaseModel, ConfigDict

# UpperCamelCase を camelCase に変換するカスタム関数
def to_lower_camel(name: str) -> str:
    """
    文字列の最初の文字を小文字に変換します。
    例: "UserName" -> "userName"
    """
    if not name:
        return ""
    return name[0].lower() + name[1:]


class User(BaseModel):
    # Pythonの定義はアッパーキャメル(UpperCamelCase)
    UserName: str
    UserID: int
    IsActive: bool = True

    # ConfigDictでカスタム関数を指定
    model_config = ConfigDict(
        alias_generator=to_lower_camel, # 自作した関数を設定
        populate_by_name=True,          # エイリアス名(camelCase)でもフィールド名(UpperCamelCase)でもデータを受け付ける
    )


# --- 動作確認 ---

# 1. フロントエンドからのリクエスト（camelCase）をパースする
request_data = {
    "userName": "Taro Yamada",
    "userID": 123,
    "isActive": False
}

# バリデーションとモデルインスタンスの作成
# Pydanticが 'userName' を 'UserName' にマッピングしてくれます
user = User.model_validate(request_data)

# Pythonコード内では UpperCamelCase でアクセス
print(f"User Name: {user.UserName}")
# -> User Name: Taro Yamada
print(f"User ID: {user.UserID}")
# -> User ID: 123


# 2. Pythonオブジェクトをレスポンス（camelCase）に変換する
# `model_dump(by_alias=True)` を使うと、エイリアス名で辞書が作成されます
response_data = user.model_dump(by_alias=True)

print("\nResponse Data:")
print(response_data)
# -> Response Data:
# -> {'userName': 'Taro Yamada', 'userID': 123, 'isActive': False}
```
