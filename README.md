MarkdownからDOCXへの変換は、おっしゃる通り構造が全く異なるため、単純な文字列置換では対応しきれません。すべての記法（ネストされたリスト、テーブル、コードブロックなど）を正確に処理するために、以下のライブラリ構成を採用します。

1. **`marked`**: Markdownを解析し、抽象構文木（AST）やHTMLに変換します。
2. **`html-to-docx`**: `marked`で生成されたHTMLを、`docx`ライブラリが解釈できる形式に変換し、Blobを生成します。
3. **`file-saver`**: 生成されたBlobをファイルとしてブラウザでダウンロードさせます。

### 1. パッケージのインストール

まず、プロジェクトのルートで以下のコマンドを実行してください。

```bash
yarn add marked html-to-docx file-saver
yarn add -D @types/marked @types/file-saver

```

---

### 2. 実装コード (React/TypeScript)

このフローでは、Markdownを一度中間形式であるHTMLに変換することで、複雑なレイアウトを維持したままDOCXへ変換します。

```typescript
import React from 'react';
import { marked } from 'marked';
import htmlToDocx from 'html-to-docx';
import { saveAs } from 'file-saver';

// html-to-docxが内部でBufferを使用するため、ブラウザ環境によって必要になる場合があります
// Vite等を使用している場合は、必要に応じてブラウザ用ポリフィルを検討してください

interface DocxDownloaderProps {
  markdownContent: string;
  fileName?: string;
}

export const MarkdownToDocxExporter: React.FC<DocxDownloaderProps> = ({ 
  markdownContent, 
  fileName = 'document.docx' 
}) => {

  /**
   * MarkdownをDOCXに変換してダウンロードするハンドラー
   */
  const handleDownload = async () => {
    if (!markdownContent) {
      console.error('変換するコンテンツが空です');
      return;
    }

    try {
      // 1. MarkdownをHTML文字列に変換
      // marked.parseはPromiseを返す場合があるためawaitを使用
      const htmlContent = await marked.parse(markdownContent);

      // 2. html-to-docxのオプション設定（フォントや余白など）
      const opt = {
        margin: {
          top: 720, // 1 inch = 1440 twips
          right: 720,
          bottom: 720,
          left: 720,
        },
        orientation: 'portrait' as const,
        font: 'Yu Mincho', // 日本語環境に適したフォントを指定
        decodeEntities: true,
      };

      // 3. HTMLからDOCXのBlobを生成
      // 第1引数は完全なHTMLドキュメント形式である必要があるためタグを付与
      const fullHtml = `<!DOCTYPE html><html><head><meta charset="utf-8"></head><body>${htmlContent}</body></html>`;
      
      const docxBlob = await htmlToDocx(fullHtml, undefined, opt);

      // 4. file-saverでダウンロード実行
      saveAs(docxBlob as Blob, fileName);

    } catch (error) {
      console.error('DOCXの生成中にエラーが発生しました:', error);
      alert('ファイルの生成に失敗しました。');
    }
  };

  return (
    <button 
      onClick={handleDownload}
      className="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700 transition-colors"
    >
      DOCXとしてダウンロード
    </button>
  );
};

```

---

### 3. 実装のポイントと注意点

* **テーブル記法への対応**: `marked`は標準でGFM (GitHub Flavored Markdown) のテーブルをサポートしているため、AIが回答した表形式も適切にHTMLの `<table>` タグに変換され、`html-to-docx`がそれをWordの表として構築します。
* **日本語フォント**: Wordファイルを開いた時の文字化けを防ぐため、オプションの `font` プロパティに `'Yu Mincho'` や `'MS Gothic'` などを指定しています。
* **環境への配慮**: `html-to-docx` は内部で一部 Node.js 由来のロジック（`Buffer`など）を使用することがあります。もし Vite をお使いで `ReferenceError: Buffer is not defined` が出た場合は、`vite-plugin-node-polyfills` を導入するか、以下を `main.tsx` 等の冒頭に記述してください。

```typescript
// Bufferのポリフィル (必要に応じて)
import { Buffer } from 'buffer';
window.Buffer = window.Buffer || Buffer;

```

この構成であれば、Markdownのリスト構造が崩れたり、リンクがただの文字列になったりするリスクを最小限に抑えつつ、TypeScriptの型安全性を保ったまま実装が可能です。

次に、このダウンロード機能に**「特定のスタイル（会社のロゴやヘッダーなど）」**を追加する方法について詳しくお伝えしましょうか？
