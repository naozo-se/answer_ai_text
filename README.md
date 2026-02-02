import React from 'react';
import { 
  Document, Packer, Paragraph, TextRun, HeadingLevel, 
  ExternalHyperlink, Table, TableRow, TableCell, 
  BorderStyle, WidthType, AlignmentType
} from 'docx';
import { saveAs } from 'file-saver';
import { unified } from 'unified';
import remarkParse from 'remark-parse';
import remarkGfm from 'remark-gfm';

export const MarkdownToDocx: React.FC<{ markdown: string }> = ({ markdown }) => {

  /**
   * インライン要素（テキスト、装飾、リンク）を解析
   */
  const processInlines = (nodes: any[], styles: { bold?: boolean; italics?: boolean; strike?: boolean } = {}): any[] => {
    return nodes.flatMap(node => {
      const currentStyles = {
        bold: styles.bold || node.type === 'strong',
        italics: styles.italics || node.type === 'emphasis',
        strike: styles.strike || node.type === 'delete'
      };

      switch (node.type) {
        case 'text':
          return [new TextRun({ text: node.value, ...currentStyles })];
        
        case 'strong':
        case 'emphasis':
        case 'delete':
          return processInlines(node.children, currentStyles);

        case 'link': {
          // 【修正の核心】リンク内のテキストノードを再帰的に取得し、TextRunインスタンスの配列を作成
          const linkChildren = processInlines(node.children, currentStyles).map(child => {
            if (child instanceof TextRun) {
              // リンク内のテキストに青色と下線を強制付与
              return new TextRun({
                text: (child as any).text,
                bold: (child as any).bold,
                italics: (child as any).italics,
                strike: (child as any).strike,
                color: '0563C1',
                underline: { color: '0563C1' }
              });
            }
            return child;
          });

          // もし中身が空ならURLをテキストとして表示
          if (linkChildren.length === 0) {
            linkChildren.push(new TextRun({ text: node.url, color: '0563C1', underline: {} }));
          }

          // ExternalHyperlink インスタンスを返す
          return [new ExternalHyperlink({
            children: linkChildren,
            link: node.url,
          })];
        }

        case 'inlineCode':
          return [new TextRun({ 
            text: node.value, 
            font: 'Courier New', 
            shading: { fill: 'EEEEEE' },
            color: 'CC0000',
            ...currentStyles
          })];

        case 'break':
          return [new TextRun({ break: 1 })];

        default:
          return node.children ? processInlines(node.children, currentStyles) : [];
      }
    });
  };

  /**
   * ブロック要素（段落、リスト、引用、水平線）を解析
   */
  const transformBlocks = (
    nodes: any[], 
    context: { level: number; ordered: boolean; quoteDepth: number } = { level: 0, ordered: false, quoteDepth: 0 }
  ): any[] => {
    const results: any[] = [];

    nodes.forEach(node => {
      switch (node.type) {
        case 'thematicBreak':
          results.push(new Paragraph({
            border: { bottom: { color: 'auto', space: 1, style: BorderStyle.SINGLE, size: 6 } },
            spacing: { before: 200, after: 200 }
          }));
          break;

        case 'heading':
          const hLevels = [HeadingLevel.HEADING_1, HeadingLevel.HEADING_2, HeadingLevel.HEADING_3, HeadingLevel.HEADING_4, HeadingLevel.HEADING_5, HeadingLevel.HEADING_6];
          results.push(new Paragraph({
            children: processInlines(node.children),
            heading: hLevels[node.depth - 1],
            spacing: { before: 240, after: 120 },
            indent: context.quoteDepth > 0 ? { left: 720 * context.quoteDepth } : undefined
          }));
          break;

        case 'paragraph':
          results.push(new Paragraph({
            children: processInlines(node.children),
            spacing: { after: 120 },
            indent: context.quoteDepth > 0 ? { left: 720 * context.quoteDepth } : undefined,
            border: context.quoteDepth > 0 ? { 
              left: { color: 'cccccc', space: 1, style: BorderStyle.SINGLE, size: 24 } 
            } : undefined
          }));
          break;

        case 'blockquote':
          results.push(...transformBlocks(node.children, { ...context, quoteDepth: context.quoteDepth + 1 }));
          break;

        case 'list':
          results.push(...transformBlocks(node.children, { ...context, ordered: node.ordered }));
          break;

        case 'listItem': {
          const immediateContentNodes: any[] = [];
          const nestedListNodes: any[] = [];
          
          node.children.forEach((child: any) => {
            if (child.type === 'list') {
              nestedListNodes.push(child);
            } else if (child.type === 'paragraph') {
              immediateContentNodes.push(...child.children);
            } else {
              immediateContentNodes.push(child);
            }
          });

          const lineChildren: any[] = [];
          if (node.checked !== null && node.checked !== undefined) {
            lineChildren.push(new TextRun({ text: node.checked ? '☑ ' : '☐ ', font: 'MS Gothic' }));
          }
          lineChildren.push(...processInlines(immediateContentNodes));

          results.push(new Paragraph({
            children: lineChildren,
            bullet: context.ordered ? undefined : { level: context.level },
            numbering: context.ordered ? { reference: 'main-numbering', level: context.level } : undefined,
            spacing: { after: 80 },
            indent: context.quoteDepth > 0 ? { left: 720 * context.quoteDepth, hanging: 360 } : undefined
          }));

          if (nestedListNodes.length > 0) {
            results.push(...transformBlocks(nestedListNodes, { ...context, level: context.level + 1 }));
          }
          break;
        }

        case 'table':
          results.push(new Table({
            width: { size: 100, type: WidthType.PERCENTAGE },
            rows: node.children.map((row: any) => new TableRow({
              children: row.children.map((cell: any) => new TableCell({
                children: [new Paragraph({ children: processInlines(cell.children) })],
                shading: node.children[0] === row ? { fill: 'F2F2F2' } : undefined,
              }))
            }))
          }));
          break;

        case 'code':
          results.push(new Paragraph({
            children: [new TextRun({ text: node.value, font: 'Courier New', size: 18 })],
            shading: { fill: 'F4F4F4' },
            indent: { left: 240 }
          }));
          break;
      }
    });

    return results;
  };

  const handleDownload = async () => {
    try {
      const processor = unified().use(remarkParse).use(remarkGfm);
      const ast = processor.parse(markdown) as any;
      const docElements = transformBlocks(ast.children);

      const doc = new Document({
        numbering: {
          config: [{
            reference: 'main-numbering',
            levels: Array.from({ length: 9 }, (_, i) => ({
              level: i, format: 'decimal', text: `%${i + 1}.`, alignment: AlignmentType.LEFT,
              style: { paragraph: { indent: { left: 720 * (i + 1), hanging: 360 } } }
            }))
          }]
        },
        sections: [{ children: docElements }]
      });

      const blob = await Packer.toBlob(doc);
      saveAs(blob, 'markdown-report.docx');
    } catch (e) {
      console.error(e);
      alert('変換中にエラーが発生しました');
    }
  };

  return <button onClick={handleDownload}>DOCXを保存</button>;
};
