function Test-StringFormat-Strict {
    param(
        [string]$InputString
    )

    # 1. まずは正規表現で大まかな形式をチェック
    if ($InputString -notmatch '^SYSTEM-\d{8}$') {
        throw "'$InputString' は形式が不正です。'SYSTEM-yyyymmdd' の形式である必要があります。"
    }

    # 2. 日付部分（8桁の数字）を抜き出す
    $datePart = $InputString.Substring(7)

    # 3. 抜き出した部分が有効な日付かチェック
    $isValidDate = [datetime]::TryParseExact(
        $datePart,
        'yyyyMMdd',
        [System.Globalization.CultureInfo]::InvariantCulture,
        [System.Globalization.DateTimeStyles]::None,
        [ref]$null
    )

    if (-not $isValidDate) {
        # 日付として無効な場合、エラーを発生させる
        throw "日付部分 '$datePart' が無効です。存在する日付を入力してください。"
    }

    # 形式が正しい場合
    Write-Host "'$InputString' は正しい形式です。"
}


# --- 実行例 ---

# 成功する例
Test-StringFormat-Strict -InputString 'SYSTEM-20251001'

# 失敗する例（あり得ない日付）
try {
    Test-StringFormat-Strict -InputString 'SYSTEM-20250230' # 2025年の2月は28日まで
}
catch {
    Write-Warning $_
}



# テスト用の文字列
$string = 'PRODUCT-123'

# -match で正規表現に一致するか評価
if ($string -match '^(?<TEMP>\w+)-(?<ALL>\d{3})') {

    Write-Host "マッチしました！"

    # 自動変数 $matches の中身を確認
    # $matches はハッシュテーブル（連想配列）になっています
    $matches | Format-Table

    # 名前を指定して、キャプチャした部分を取り出す
    Write-Host "TEMP の値: $($matches.TEMP)" # ← 'PRODUCT' が取り出せる
    Write-Host "ALL の値: $($matches.ALL)"   # ← '123' が取り出せる
}
