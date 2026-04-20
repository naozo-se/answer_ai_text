import io
from pptx import Presentation
from PIL import Image
import google.generativeai as genai

def extract_pptx_elements(file_path: str) -> list:
    """
    PPTXファイルを解析し、テキストと画像を順番通りに抽出して
    Gemini APIへ渡すためのリスト（プロンプトパーツ）に変換する関数。
    """
    prs = Presentation(file_path)
    prompt_parts = []

    for i, slide in enumerate(prs.slides):
        slide_text = ""
        slide_images = []

        # 1. スライド内の要素（シェイプ）を走査
        for shape in slide.shapes:
            # テキストの抽出
            if hasattr(shape, "text") and shape.text.strip():
                slide_text += shape.text + "\n"
            
            # 画像(PICTURE)の抽出 (shape_type == 13)
            if shape.shape_type == 13:
                image_bytes = shape.image.blob
                img = Image.open(io.BytesIO(image_bytes))
                slide_images.append(img)

        # 2. スライドごとの情報をAPIに渡す形式でリスト化
        prompt_parts.append(f"--- Slide {i+1} ---")
        
        if slide_text:
            prompt_parts.append(f"Text:\n{slide_text.strip()}")
        
        # 画像オブジェクトをそのままリストに追加（Gemini APIはPIL.Imageを直接解釈可能）
        prompt_parts.extend(slide_images)

    return prompt_parts


# ==========================================
# 実行部分（メインのロジック）
# ==========================================
def main():
    # APIキーと最新のモデルを指定
    genai.configure(api_key="YOUR_GEMINI_API_KEY")
    
    # ※ご利用の環境に合わせて最新のモデル名（gemini-1.5-pro, gemini-2.0-flashなど）を指定してください
    model = genai.GenerativeModel('gemini-1.5-pro')

    file_path = "presentation.pptx"
    
    # 1. 切り出した関数でファイルを解析し、マルチモーダル入力用の配列を取得
    contents = extract_pptx_elements(file_path)
    
    # 2. ユーザーの指示（プロンプト）を配列の末尾に追加
    contents.append("上記のプレゼンテーション資料（テキストと画像）を総合して、内容を要約してください。")
    
    # 3. APIへリクエスト
    print("AIにリクエストを送信中...")
    response = model.generate_content(contents)
    
    print("\n【AIの回答】")
    print(response.text)

if __name__ == "__main__":
    main()
