承知いたしました。表（Table）を利用するとセルの余白やレイアウトの制御が複雑になりがちですので、**段落（Paragraph）の境界線とインデントを組み合わせた元の方式**に戻し、さらに**ドキュメント全体のデフォルトフォントを一括設定**するコードを作成しました。

このプロジェクトでは、TypeScriptとReact環境を使用して、Markdownから`.docx`ファイルへの正確な変換を目指しています。

### 1. デフォルトフォントの設定方法

`docx` ライブラリでは、`Document` インスタンスを作成する際の `styles` プロパティでデフォルトのフォントやサイズを指定できます。

* **font**: フォント名（例: "Yu Gothic", "MS Gothic", "Arial"）
* **size**: フォントサイズ（**半ポイント単位**。10.5ptにしたい場合は `21`、12ptなら `24` を指定）

### 2. 修正版実装コード

引用（`blockquote`）を段落境界線方式に戻し、全体のデフォルトスタイルを適用した構成です。

```tsx
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
   * インライン要素（テキスト、装飾、リンク）の解析
   */
  const processInlines = (nodes: any[], styles: { bold?: boolean; italics?: boolean; strike?: boolean; color?: string; underline?: any } = {}): any[] => {
    return nodes.flatMap(node => {
      const currentStyles = {
        bold: styles.bold || node.type === 'strong',
        italics: styles.italics || node.type === 'emphasis',
        strike: styles.strike || node.type === 'delete',
        color: styles.color,
        underline: styles.underline
      };

      switch (node.type) {
        case 'text':
          const lines = node.value.split('\n');
          return lines.map((line: string, i: number) => new TextRun({ 
            text: line, 
            break: i > 0 ? 1 : undefined, 
            ...currentStyles 
          }));
        
        case 'strong':
        case 'emphasis':
        case 'delete':
          return processInlines(node.children, currentStyles);

        case 'link': {
          const linkTextRuns = processInlines(node.children, {
            ...currentStyles,
            color: '0563C1',
            underline: { color: '0563C1' }
          }).filter(child => child instanceof TextRun);

          return [new ExternalHyperlink({
            children: linkTextRuns,
            link: node.url || '',
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

        default:
          return node.children ? processInlines(node.children, currentStyles) : [];
      }
    });
  };

  /**
   * ブロック要素の解析（引用は段落境界線方式に復帰）
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
          results.push(new Paragraph({
            children: processInlines(node.children),
            heading: [HeadingLevel.HEADING_1, HeadingLevel.HEADING_2, HeadingLevel.HEADING_3, HeadingLevel.HEADING_4, HeadingLevel.HEADING_5, HeadingLevel.HEADING_6][node.depth - 1] || HeadingLevel.HEADING_1,
            spacing: { before: 240, after: 120 },
            indent: context.quoteDepth > 0 ? { left: 720 * context.quoteDepth } : undefined
          }));
          break;

        case 'paragraph':
          results.push(new Paragraph({
            children: processInlines(node.children),
            spacing: { after: 120 },
            // 引用の深さに応じて左線とインデントを適用
            indent: context.quoteDepth > 0 ? { left: 720 * context.quoteDepth } : undefined,
            border: context.quoteDepth > 0 ? { 
              left: { color: 'cccccc', space: 1, style: BorderStyle.SINGLE, size: 24 } 
            } : undefined
          }));
          break;

        case 'blockquote':
          // 再帰的に中身を処理し、quoteDepthを加算
          results.push(...transformBlocks(node.children, { ...context, quoteDepth: context.quoteDepth + 1 }));
          break;

        case 'list':
          results.push(...transformBlocks(node.children, { ...context, ordered: node.ordered }));
          break;

        case 'listItem': {
          const immediateContentNodes: any[] = [];
          const nestedListNodes: any[] = [];
          node.children.forEach((c: any) => (c.type === 'list' ? nestedListNodes : immediateContentNodes).push(c));

          const lineChildren: any[] = [];
          if (node.checked !== null && node.checked !== undefined) {
            lineChildren.push(new TextRun({ text: node.checked ? '☑ ' : '☐ ', font: 'MS Gothic' }));
          }
          
          const flatInlines = immediateContentNodes.flatMap(c => c.type === 'paragraph' ? c.children : [c]);
          lineChildren.push(...processInlines(flatInlines));

          results.push(new Paragraph({
            children: lineChildren,
            bullet: context.ordered ? undefined : { level: context.level },
            numbering: context.ordered ? { reference: 'main-numbering', level: context.level } : undefined,
            spacing: { after: 80 },
            indent: { 
              left: (context.quoteDepth * 720) + (context.level * 360) + 360, 
              hanging: 360 
            }
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
      const ast = unified().use(remarkParse).use(remarkGfm).parse(markdown) as any;
      const docElements = transformBlocks(ast.children);

      const doc = new Document({
        // デフォルトフォントの設定
        styles: {
          default: {
            paragraph: {
              run: {
                font: "Yu Gothic", // 游ゴシック
                size: 21,          // 10.5pt
              },
            },
          },
        },
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
      alert('変換中にエラーが発生しました。');
    }
  };

  return <button onClick={handleDownload}>DOCXをダウンロード</button>;
};

```

---

### 今回の変更点

* **デフォルトフォントの適用**: `Document` の `styles` セクションに、標準的な「游ゴシック（Yu Gothic）」と、日本で一般的な「10.5pt（サイズ: 21）」を適用しました。
* **引用スタイルの差し戻し**: 表（Table）を廃止し、`Paragraph` の `border` と `indent` プロパティで垂直線を表現する方式に戻しました。これにより、Word標準の段落としての挙動が維持されます。
* **構造の整理**: 2026年現在の開発プロジェクトにおいて、正確なMarkdown解析とスタイリングを両立させるためのロジックを担保しています。

ドキュメントの「ページ余白（マージン）」や「ヘッダー・フッター」の追加についても設定可能ですが、あわせて実装いたしますか？
