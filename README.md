マークダウンを一度HTMLに変換し、それをWord（docx）として出力するアプローチですね。構造的なロジックを組む方法とは別に、既存のHTMLレンダリングを活かせるため、レイアウトの再現性が期待できる手法です。

ご提案の `@turbodocx/html-to-docx` を利用し、マークダウン解析にはデファクトスタンダードである `marked` を組み合わせた実装例を紹介します。

### 1. パッケージのインストール

まず、必要なライブラリを `yarn` でインストールします。

```bash
yarn add @turbodocx/html-to-docx marked file-saver
yarn add -D @types/marked @types/file-saver

```

---

### 2. 実装コード (React用ハンドラー)

このコードは、マークダウン文字列を受け取り、HTMLへ変換した後、BlobとしてWordファイルを生成・ダウンロードする一連の流れを完結させます。

```typescript
import React from 'react';
import { marked } from 'marked';
import { saveAs } from 'file-saver';
import htmlToDocx from '@turbodocx/html-to-docx';

export const useDocxDownloader = () => {
  /**
   * MarkdownをDocxに変換してダウンロードするハンドラー
   * @param markdownText AIから返却されたマークダウン文字列
   * @param fileName 保存時のファイル名
   */
  const handleDownloadDocx = async (markdownText: string, fileName: string = 'document.docx') => {
    try {
      // 1. MarkdownをHTMLに変換
      // marked.parse は string | Promise<string> を返却するため await 
      const htmlContent = await marked.parse(markdownText);

      // 2. html-to-docx 用のオプション設定
      // 必要に応じてフォントや余白を調整できます
      const opt = {
        margin: {
          top: 720,    // 1 inch
          right: 720,
          bottom: 720,
          left: 720,
        },
        header: true,
        footer: true,
        pageNumber: true,
      };

      // 3. HTMLからDocx(Blob)を生成
      // 第1引数は完全なHTML構造である必要があるため、bodyタグなどでラップします
      const fullHtml = `<!DOCTYPE html><html><body>${htmlContent}</body></html>`;
      const docxBlob = await htmlToDocx(fullHtml, null, opt);

      // 4. file-saverでダウンロード実行
      saveAs(docxBlob, fileName);

    } catch (error) {
      console.error('Docxの生成に失敗しました:', error);
      alert('ファイルの作成中にエラーが発生しました。');
    }
  };

  return { handleDownloadDocx };
};

// --- 利用側のコンポーネント例 ---
const MyComponent = () => {
  const { handleDownloadDocx } = useDocxDownloader();
  
  // AIの回答などを想定したサンプルマークダウン
  const aiResponse = `
# システム仕様書
## 概要
これは**React**で生成されたドキュメントです。
- リスト項目1
- リスト項目2
  `;

  return (
    <button onClick={() => handleDownloadDocx(aiResponse, 'specification.docx')}>
      Wordでダウンロード
    </button>
  );
};

```

---

### 実装のポイント

* **HTMLのラップ:** `htmlToDocx` は、渡された文字列をHTMLドキュメントとして解釈します。`marked` で生成されたフラグメント（`<h1>...</h1>`など）を `<html><body>...</body></html>` で囲うことで、パースミスを防ぎ、より安定した出力を得られます。
* **非同期処理:** HTMLからWordへの変換は重い処理になる可能性があるため、`async/await` で非同期に処理し、UIをブロックしないようにしています。
* **スタイリングの拡張:** `@turbodocx/html-to-docx` は、インラインスタイル（`<span style="color: red">`など）をある程度解釈してWordの書式に落とし込んでくれます。もし特定のデザインが必要な場合は、`marked` のレンダラーをカスタマイズして特定のタグにスタイルを付与する運用がスムーズです。

このアプローチなら、構造を1つずつ `docx` ライブラリのオブジェクト（`Paragraph`, `TextRun` など）にマッピングする手間が省けるため、メンテナンス性も高まるはずです。

もし、特定のCSSスタイル（表の罫線やフォント指定など）を反映させたい場合は、その追加設定についてもお手伝いできますがいかがでしょうか？
