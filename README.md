はい、もちろんです。処理を関数化すると、繰り返し利用しやすくなり、コードも整理されるのでおすすめです。

前のスクリプトを元に、より柔軟で再利用しやすい関数を作成しました。

-----

### TIFFからJPGへ変換する関数

以下は、指定されたTIFFファイル（またはフォルダ内の全TIFFファイル）をJPGに変換する関数 `Convert-TiffToJpeg` です。

```powershell
function Convert-TiffToJpeg {
    [CmdletBinding()]
    param(
        # 変換したいTIFFファイル、またはTIFFファイルが含まれるフォルダのパスを指定します。
        [Parameter(Mandatory=$true, Position=0, ValueFromPipeline=$true)]
        [string[]]$Path,

        # 変換後のJPGファイルを保存するフォルダのパスを指定します。
        [Parameter(Mandatory=$true)]
        [string]$OutputFolder,

        # フォルダを指定した際に、サブフォルダ内も再帰的に検索する場合に指定します。
        [Parameter()]
        [switch]$Recurse
    )

    begin {
        # .NETアセンブリを最初に一度だけ読み込む
        try {
            Add-Type -AssemblyName System.Drawing
        }
        catch {
            Write-Error "System.Drawingアセンブリの読み込みに失敗しました。処理を続行できません。"
            return
        }

        # 出力フォルダが存在しない場合は作成する
        if (-not (Test-Path $OutputFolder)) {
            Write-Verbose "出力フォルダ '$OutputFolder' を作成します。"
            New-Item -ItemType Directory -Path $OutputFolder | Out-Null
        }
    }

    process {
        foreach ($itemPath in $Path) {
            # パスで指定されたアイテム情報を取得
            $item = Get-Item -Path $itemPath -ErrorAction SilentlyContinue
            if (-not $item) {
                Write-Warning "指定されたパスが見つかりません: $itemPath"
                continue
            }

            # 対象となるファイルを取得
            $filesToProcess = @()
            if ($item -is [System.IO.DirectoryInfo]) {
                # フォルダの場合、中のtiff/tifファイルを取得
                $filesToProcess += Get-ChildItem -Path $item.FullName -Include "*.tiff", "*.tif" -Recurse:$Recurse
            }
            elseif ($item.Extension -in ".tiff", ".tif") {
                # ファイルの場合、そのファイル自身を対象に
                $filesToProcess += $item
            }

            # ファイルを一つずつ変換
            foreach ($file in $filesToProcess) {
                $jpegFileName = [System.IO.Path]::ChangeExtension($file.Name, ".jpg")
                $jpegFilePath = Join-Path $OutputFolder $jpegFileName

                try {
                    $image = [System.Drawing.Image]::FromFile($file.FullName)
                    $image.Save($jpegFilePath, [System.Drawing.Imaging.ImageFormat]::Jpeg)
                    $image.Dispose()
                    Write-Host "変換完了: $($file.FullName) -> $($jpegFilePath)"
                }
                catch {
                    Write-Error "エラー: $($file.FullName) の変換中に問題が発生しました。 - $($_.Exception.Message)"
                }
            }
        }
    }
}
```

-----

### 関数の使い方

この関数をPowerShellで使うには、いくつかの方法があります。

1.  **PowerShellの画面に直接貼り付け**
    上記のコードをコピーし、PowerShellのウィンドウに貼り付けてEnterキーを押すと、そのセッションで `Convert-TiffToJpeg` 関数が使えるようになります。

2.  **スクリプトファイルとして保存**
    コードを `Convert-TiffToJpeg.ps1` のような名前で保存し、利用する際にそのファイルを読み込みます。

    ```powershell
    # スクリプトファイルを読み込んで関数を定義する (ドットソーシング)
    . C:\path\to\your\script\Convert-TiffToJpeg.ps1
    ```

#### 実行例

関数を定義した後、以下のように使えます。

**例1: 単一のファイルを変換する**

```powershell
Convert-TiffToJpeg -Path "C:\images\photo.tiff" -OutputFolder "C:\output"
```

**例2: フォルダ内の全TIFFファイルを変換する**

```powershell
Convert-TiffToJpeg -Path "C:\images" -OutputFolder "C:\output"
```

**例3: フォルダとサブフォルダ内の全TIFFファイルを変換する (`-Recurse` を使用)**

```powershell
Convert-TiffToJpeg -Path "C:\images" -OutputFolder "C:\output" -Recurse
```

**例4: パイプラインを使ってファイルを渡す（よりPowerShellらしい使い方）**
`Get-ChildItem` で取得したファイルオブジェクトを直接パイプライン `|` で関数に渡すこともできます。

```powershell
Get-ChildItem -Path "C:\images\*.tiff" | Convert-TiffToJpeg -OutputFolder "C:\output"
```
