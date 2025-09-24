# ----------------------------------------------------
# ステップ1：ここに関数の定義を記述します
# ----------------------------------------------------
function Convert-TiffToJpg {
    <#
    .SYNOPSIS
        指定されたフォルダ内のTIFFファイルをJPGに変換します。
    .DESCRIPTION
        入力フォルダと出力フォルダは同じです。.tif と .tiff の両方の拡張子に対応します。
    .PARAMETER FolderPath
        TIFFファイルが含まれるフォルダのパスを指定します。
    .EXAMPLE
        Convert-TiffToJpg -FolderPath "C:\Users\YourUser\Desktop\Images"
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true, ValueFromPipeline = $true, Position = 0)]
        [string]$FolderPath
    )

    # .NETのSystem.Drawingアセンブリを読み込む
    try {
        Add-Type -AssemblyName System.Drawing
    }
    catch {
        Write-Error "エラー: .NETの System.Drawing アセンブリを読み込めませんでした。"
        return
    }

    # 指定されたパスがフォルダとして存在するか確認
    if (-not (Test-Path -Path $FolderPath -PathType Container)) {
        Write-Error "エラー: 指定されたパス '$FolderPath' は存在しないか、フォルダではありません。"
        return
    }

    # フォルダ内のTIFFファイルを取得
    $tiffFiles = Get-ChildItem -Path "$FolderPath\*" -Include @("*.tif", "*.tiff")

    if ($tiffFiles.Count -eq 0) {
        Write-Warning "指定されたフォルダにTIFFファイルが見つかりませんでした。"
        return
    }

    Write-Host "変換処理を開始します。対象フォルダ: $FolderPath"

    foreach ($file in $tiffFiles) {
        $outputFilePath = [System.IO.Path]::ChangeExtension($file.FullName, ".jpg")
        $image = $null

        try {
            $image = [System.Drawing.Bitmap]::new($file.FullName)
            $image.Save($outputFilePath, [System.Drawing.Imaging.ImageFormat]::Jpeg)
            Write-Host "変換成功: $($file.Name) -> $(Split-Path $outputFilePath -Leaf)"
        }
        catch {
            Write-Error "ファイル変換中にエラーが発生しました: $($file.Name) - $($_.Exception.Message)"
        }
        finally {
            if ($image -ne $null) {
                $image.Dispose()
            }
        }
    }

    Write-Host "すべての処理が完了しました。"
}
