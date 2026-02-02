引用の改行がまとまってしまう問題は、Markdownパーサー（remark）が「連続する引用行」を一つの段落（Paragraph）として結合し、その中の改行コード（`\n`）を無視して処理していたことが原因です。

これを解決するために、**テキスト内の改行コードを明示的にWordの「改行（Break）」に変換するロジック**を組み込みました。あわせて、引用やリストのインデント計算をさらに精密に調整しています。

### 修正版：引用の改行と階層構造を維持する完全コード

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
   * テキスト内の改行コード（\n）をWordの改行に変換する処理を追加
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
          // 【核心の修正】テキスト内の \n を分割し、2行目以降に break: 1 を付与してTextRunを生成
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

        case 'break': // Markdownの明示的な改行（スペース2つ + 改行）
          return [new TextRun({ break: 1 })];

        default:
          return node.children ? processInlines(node.children, currentStyles) : [];
      }
    });
  };

  /**
   * ブロック要素の解析
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
            // 引用(quoteDepth)がある場合は左線とインデントを適用
            indent: context.quoteDepth > 0 ? { left: 720 * context.quoteDepth } : undefined,
            border: context.quoteDepth > 0 ? { 
              left: { color: 'cccccc', space: 1, style: BorderStyle.SINGLE, size: 24 } 
            } : undefined
          }));
          break;

        case 'blockquote':
          // 引用ノードの中身を再帰処理。quoteDepthを増やすことで2重引用(>>)に対応。
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
            // リストが引用内にある場合のインデント調整
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
      saveAs(blob, 'final-report.docx');
    } catch (e) {
      console.error(e);
      alert('エラーが発生しました');
    }
  };

  return <button onClick={handleDownload}>DOCXをダウンロード</button>;
};

```

---

### 今回の修正ポイント解説

* **引用内の改行維持**:
`processInlines` 内の `case 'text'` を見てください。Markdownパーサーが一つにまとめてしまったテキストを `\n` で分割し、2行目以降の `TextRun` に `break: 1` を付与するようにしました。これにより、Word上でも一行ずつ改行されて表示されます。
* **ネストした引用（2重引用）のスタイル**:
`context.quoteDepth` に基づいて、左側の境界線とインデントを動的に計算するようにしています。画像にあったような「引用の中に引用がある」状態も、Wordのインデント機能で再現されます。
* **インデント計算の精密化**:
リスト項目（`listItem`）が引用内にある場合、引用の深さとリストのレベルを合算して余白を計算するように修正しました。これにより、文字が重なったり境界線を突き破ったりするのを防いでいます。

これで、リンク、取り消し線、地平線、そして「引用内の改行」まで全て網羅できました。

実際にこのコードで出力してみて、文字サイズやフォント（例えば「MS ゴシック」ではなく「游明朝」にしたい、など）の調整は必要ですか？それとも、これで運用できそうでしょうか？
