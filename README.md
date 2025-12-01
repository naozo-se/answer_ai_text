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
