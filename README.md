HTML変換を介さず、Markdownの抽象構文木（AST）を直接解析して `docx` のオブジェクトモデルに変換するアプローチで実装します。

この方法には、Markdownパーサーとして非常に強力な **`remark`** エコシステムを使用します。これにより、HTMLを経由せずに正確な構造解析が可能です。

### 1. パッケージのインストール

まず、必要なライブラリをインストールしてください。

```bash
yarn add docx file-saver remark-parse unified
yarn add -D @types/file-saver

```

### 2. 実装コード (React/TypeScript)

このコードは、Markdown文字列を `unified` / `remark-parse` で解析し、各ノード（見出し、リスト、強調など）を `docx` のコンポーネントに再帰的に変換します。

```tsx
import React from 'react';
import { 
  Document, 
  Packer, 
  Paragraph, 
  TextRun, 
  HeadingLevel, 
  ExternalHyperlink, 
  ListParagraph,
  AlignmentType,
  LevelFormat
} from 'docx';
import { saveAs } from 'file-saver';
import { unified } from 'unified';
import remarkParse from 'remark-parse';

// --- 型定義 ---
interface MarkdownNode {
  type: string;
  value?: string;
  children?: MarkdownNode[];
  depth?: number;
  url?: string;
  ordered?: boolean;
}

export const DocxDownloadButton: React.FC<{ markdownContent: string }> = ({ markdownContent }) => {

  /**
   * MarkdownのASTノードをdocxの要素に変換する再帰関数
   */
  const transformNode = (node: MarkdownNode, listLevel: number = 0): any => {
    switch (node.type) {
      case 'root':
        return node.children?.flatMap(child => transformNode(child, listLevel)) || [];

      case 'heading': {
        const level = node.depth === 1 ? HeadingLevel.HEADING_1 :
                      node.depth === 2 ? HeadingLevel.HEADING_2 :
                      HeadingLevel.HEADING_3;
        return new Paragraph({
          children: node.children?.flatMap(child => transformTextRun(child)) || [],
          heading: level,
          spacing: { before: 240, after: 120 },
        });
      }

      case 'paragraph':
        return new Paragraph({
          children: node.children?.flatMap(child => transformTextRun(child)) || [],
          spacing: { after: 120 },
        });

      case 'list':
        return node.children?.flatMap(child => transformNode(child, listLevel + 1)) || [];

      case 'listItem':
        // リストアイテム内の段落を処理
        return node.children?.flatMap(child => {
          const transformed = transformNode(child, listLevel);
          if (transformed instanceof Paragraph) {
            return new Paragraph({
              ...transformed,
              bullet: { level: listLevel - 1 }, // docxのリストレベルは0から
            });
          }
          return transformed;
        });

      case 'code': // ブロックコード
        return new Paragraph({
          children: [new TextRun({ text: node.value || '', font: 'Courier New' })],
          shading: { fill: 'F4F4F4' },
          indent: { left: 720 },
        });

      case 'blockquote':
        return node.children?.flatMap(child => {
          const transformed = transformNode(child, listLevel);
          if (transformed instanceof Paragraph) {
             return new Paragraph({
               ...transformed,
               indent: { left: 720 },
               border: { left: { color: 'cccccc', space: 1, style: 'single', size: 6 } }
             });
          }
          return transformed;
        });

      default:
        return [];
    }
  };

  /**
   * テキスト装飾（太字、斜体、インラインコード、リンク）をTextRunに変換
   */
  const transformTextRun = (node: MarkdownNode): any => {
    if (node.type === 'text') {
      return new TextRun({ text: node.value || '' });
    }
    if (node.type === 'strong') {
      return node.children?.map(child => new TextRun({ text: child.value || '', bold: true })) || [];
    }
    if (node.type === 'emphasis') {
      return node.children?.map(child => new TextRun({ text: child.value || '', italics: true })) || [];
    }
    if (node.type === 'inlineCode') {
      return new TextRun({ text: node.value || '', font: 'Courier New', shading: { fill: 'EEEEEE' } });
    }
    if (node.type === 'link') {
      return new ExternalHyperlink({
        children: [new TextRun({ text: node.children?.[0]?.value || node.url || '', color: '0563C1', underline: {} })],
        link: node.url || '',
      });
    }
    if (node.children) {
      return node.children.flatMap(child => transformTextRun(child));
    }
    return [];
  };

  /**
   * onClickハンドラー
   */
  const handleDownloadDocx = async () => {
    try {
      // 1. Markdownを解析してAST(Abstract Syntax Tree)を取得
      const processor = unified().use(remarkParse);
      const ast = processor.parse(markdownContent) as MarkdownNode;

      // 2. ASTをdocxの段落オブジェクトに変換
      const docElements = transformNode(ast);

      // 3. ドキュメント作成
      const doc = new Document({
        sections: [{
          properties: {},
          children: docElements,
        }],
      });

      // 4. Blobに変換してダウンロード
      const blob = await Packer.toBlob(doc);
      saveAs(blob, 'ai-response.docx');
    } catch (error) {
      console.error('DOCXの生成に失敗しました:', error);
      alert('ファイルの生成中にエラーが発生しました。');
    }
  };

  return (
    <button 
      onClick={handleDownloadDocx}
      className="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700"
    >
      DOCXをダウンロード
    </button>
  );
};

export default DocxDownloadButton;

```

---

### この実装のポイント

* **HTML完全不使用:** `remark-parse` を使って Markdown 文字列を直接 JavaScript オブジェクト（AST）に分解しています。そのため、ブラウザの DOM や `dangerouslySetInnerHTML` などに依存せず、セキュアかつ正確に構造を把握できます。
* **再帰的処理:** Markdown の「ネストされたリスト」や「引用の中にリスト」といった複雑な構造も、`transformNode` 関数が再帰的に処理することで `docx` の構造にマッピングしています。
* **インラインスタイルの維持:** `strong`（太字）、`emphasis`（斜体）、`inlineCode`（等幅フォント）、`link`（ハイパーリンク）を `TextRun` として適切に処理しています。
* **docx v8/v9 対応:** `docx` ライブラリの最新の書き方（`Document` の `sections` プロパティなど）に準拠しています。

### 注意点

Markdown の表（Table）に関しては、`remark-parse` 標準ではサポートされておらず、別途 `remark-gfm` などのプラグインと、それに対応する `docx` の `Table` オブジェクトへの変換ロジックを追加する必要があります。もし表の対応も必須であれば、追加で実装可能です。

このコードで、AIからの回答（マークダウン）を安全かつ綺麗にWordファイルへ変換できるはずです。他に調整したいスタイル（フォントサイズや余白など）があれば教えてくださいね。
