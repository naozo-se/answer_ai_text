
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
