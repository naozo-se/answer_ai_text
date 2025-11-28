```
(Get-Content "ファイルのパス" -Raw).Replace("`r`n","`n") | Set-Content "ファイルのパス" -NoNewline
```
