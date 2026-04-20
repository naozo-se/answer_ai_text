`.docx` と `.xlsx` からテキストと画像を抽出し、Gemini API に渡すためのプロンプトパーツ（リスト）に変換する関数ですね。

WordとExcelは内部構造がPowerPointと異なるため、要素の取得方法が少し変わります。実用的な抽出関数の実装例を作成しました。

### 必要なパッケージの追加
依存関係の管理に追加をお願いします。
```bash
poetry add python-docx openpyxl pillow
```

### 抽出関数の実装

```python
import io
import docx
import openpyxl
from PIL import Image

def _extract_docx_elements(file_path: str) -> list:
    """
    Word (.docx) ファイルからテキストと画像を抽出する関数。
    ※テキストを順番に読み込んだ後、ドキュメント内の全画像を末尾に追加するアプローチです。
    """
    doc = docx.Document(file_path)
    prompt_parts = []
    full_text = []

    # 1. テキストの抽出（段落ごと）
    for para in doc.paragraphs:
        text = para.text.strip()
        if text:
            full_text.append(text)
            
    # 表（テーブル）内のテキスト抽出
    for table in doc.tables:
        for row in table.rows:
            row_data = [cell.text.strip() for cell in row.cells if cell.text.strip()]
            if row_data:
                full_text.append(" | ".join(row_data))

    if full_text:
        prompt_parts.append("Text:\n" + "\n".join(full_text))

    # 2. 画像の抽出
    # Wordの内部構造(XML)から、画像リレーションを直接取得して安全に抽出します
    for rel in doc.part.rels.values():
        if "image" in rel.reltype:
            image_bytes = rel.target_part.blob
            img = Image.open(io.BytesIO(image_bytes))
            prompt_parts.append(img)

    return prompt_parts


def _extract_xlsx_elements(file_path: str) -> list:
    """
    Excel (.xlsx) ファイルからシートごとにテキストと画像を抽出する関数。
    """
    # 数式ではなく計算結果の値を取得するために data_only=True を指定
    wb = openpyxl.load_workbook(file_path, data_only=True)
    prompt_parts = []

    for sheet in wb.worksheets:
        prompt_parts.append(f"--- Sheet: {sheet.title} ---")
        sheet_text = []

        # 1. セルのテキスト抽出（行ごと）
        for row in sheet.iter_rows(values_only=True):
            # Noneを除外して文字列化
            row_data = [str(cell) for cell in row if cell is not None]
            if row_data:
                sheet_text.append(" | ".join(row_data))

        if sheet_text:
            prompt_parts.append("Text:\n" + "\n".join(sheet_text))

        # 2. 画像の抽出
        # openpyxlの内部プロパティからシートに紐づく画像を抽出します
        for image in getattr(sheet, '_images', []):
            # openpyxlの画像データからPillowのImageオブジェクトを生成
            image_bytes = image._data()
            img = Image.open(io.BytesIO(image_bytes))
            prompt_parts.append(img)

    return prompt_parts
```

### 仕様上の注意点（Word / Excelのパースにおける限界）

このアプローチには、データ構造に起因するいくつかのトレードオフがあります。

* **Word (.docx) の順序性:** 上記のコードでは、抽出の確実性を優先し「ドキュメント内の全テキスト」のあとに「全画像」をまとめてプロンプトパーツに追加しています。テキスト内の特定の段落とインライン画像の位置関係を厳密に維持するには、生XMLのパースが必要になり、保守性が著しく低下するためです。
* **Excel (.xlsx) のアンカー（配置）:**
  Excelの画像はセルの中に存在するのではなく、シートというキャンバスの上に「浮いている（Floating）」状態です。そのため、上記の関数では「どのセルの近くにどの画像があったか」というレイアウト情報は失われ、シート単位でのコンテキスト渡しとなります。

これらの抽出処理は、ドキュメントから抽出したデータをRAGなどの知識ベースに投入するための前処理として検討されているのでしょうか？
