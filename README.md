大詰めですね！ご指摘いただいた「取り消し線」「リンクの消失」「水平線」について、`remark-gfm` が出力する特定のノード型（`delete`, `link`, `thematicBreak`）を `docx` のプロパティに正しくマッピングするよう修正しました。

特にリンクについては、`docx` ライブラリの仕様上、`ExternalHyperlink` が `Paragraph` の直下にある必要があるため、その構造を担保するように調整しています。

### 修正版：Markdown to DOCX 変換コード

```tsx
import React from 'react';
import { 
  Document, Packer, Paragraph, TextRun, HeadingLevel, 
  ExternalHyperlink, Table, TableRow, TableCell, 
  BorderStyle, WidthType, AlignmentType, ThematicBreak
} from 'docx';
import { saveAs } from 'file-saver';
import { unified } from 'unified';
import remarkParse from 'remark-parse';
import remarkGfm from 'remark-gfm';

export const MarkdownToDocx: React.FC<{ markdown: string }> = ({ markdown }) => {

  /**
   * インライン要素の解析
   * 取り消し線(strike)とリンク(link)の処理を強化
   */
  const processInlines = (nodes: any[], styles: { bold?: boolean; italics?: boolean; strike?: boolean } = {}): any[] => {
    return nodes.flatMap(node => {
      const currentStyles = {
        bold: styles.bold || node.type === 'strong',
        italics: styles.italics || node.type === 'emphasis',
        strike: styles.strike || node.type === 'delete' // 取り消し線の対応
      };

      switch (node.type) {
        case 'text':
          return new TextRun({ text: node.value, ...currentStyles });
        
        case 'strong':
        case 'emphasis':
        case 'delete': // 取り消し線ノード
          return processInlines(node.children, currentStyles);

        case 'link':
          // リンクタイトルも装飾を維持
          return new ExternalHyperlink({
            children: processInlines(node.children, currentStyles).map(run => {
              if (run instanceof TextRun) {
                // リンクらしく青色・下線をつける
                return new TextRun({ ...run, color: '0563C1', underline: { color: '0563C1' } });
              }
              return run;
            }),
            link: node.url,
          });

        case 'inlineCode':
          return new TextRun({ 
            text: node.value, 
            font: 'Courier New', 
            shading: { fill: 'EEEEEE' },
            color: 'CC0000',
            ...currentStyles
          });

        case 'break':
          return new TextRun({ break: 1 });

        default:
          return node.children ? processInlines(node.children, currentStyles) : [];
      }
    });
  };

  /**
   * ブロック要素の解析
   * 水平線(thematicBreak)の対応を追加
   */
  const transformBlocks = (
    nodes: any[], 
    context: { level: number; ordered: boolean; quoteDepth: number } = { level: 0, ordered: false, quoteDepth: 0 }
  ): any[] => {
    const results: any[] = [];

    nodes.forEach(node => {
      switch (node.type) {
        case 'thematicBreak': // --- (水平線) の対応
          results.push(new Paragraph({
            border: {
              bottom: { color: 'auto', space: 1, style: BorderStyle.SINGLE, size: 6 }
            },
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
            indent: context.quoteDepth > 0 ? { left: 720 * context.quoteDepth + (context.level * 360), hanging: 360 } : undefined
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
      alert('変換エラーが発生しました');
    }
  };

  return <button onClick={handleDownload} className="btn">保存</button>;
};

```

---

### 今回の修正ポイント解説

1. **取り消し線 (Strikethrough)**:
* `remark-gfm` は `~~text~~` を `delete` というノード型で出力します。
* `processInlines` 内で `delete` を検知し、`TextRun` のプロパティに `{ strike: true }` をセットすることで、Word上でも取り消し線が表示されるようにしました。


2. **リンクの完全な復元**:
* 以前のコードでリンクが表示されなかったのは、`ExternalHyperlink` の `children` が正しく渡されていなかったか、型判定で弾かれていた可能性があります。
* リンク内のテキストに対しても `processInlines` を再帰的に適用し、色と下線を明示的に指定した `TextRun` を持たせることで確実に表示されるようにしました。


3. **水平線 (`---`)**:
* Markdownの `---` は `thematicBreak` ノードになります。
* Wordには「水平線」という単一のオブジェクトもありますが、段落の下境界線（`border: { bottom: ... }`）として実装するのが最もレイアウトが崩れにくく一般的であるため、その方式を採用しました。



これで、懸念されていた「リンク」「取り消し線」「水平線」がすべて DOCX に反映されるはずです！

もし「リンクのURLもテキストとして横に表示したい」や「水平線の太さを変えたい」などの細かなデザイン調整があれば、すぐに対応可能です。いかがでしょうか？
