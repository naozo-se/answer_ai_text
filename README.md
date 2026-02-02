画像で示していただいたような、引用の深さに応じて垂直線が増えていく「ネストされた引用」をWord上で再現するには、**Wordの「表（Table）」機能を引用の枠組みとして利用する**のが最も確実で美しい方法です。

Wordの標準的な段落境界線では、一つの段落に対して一本の左線しか引けませんが、表を入れ子（ネスト）にすることで、画像通りの多重線を表現できます。

### 修正後のコード (React/TypeScript)

引用（`blockquote`）の処理をテーブル形式に刷新し、その他の改善点もすべて盛り込んだ最終版です。

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
   * インライン要素の解析
   * テキスト、装飾、リンク、および改行の処理
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
          // 改行コード \n を分割して Word の break に変換
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

        case 'break':
          return [new TextRun({ break: 1 })];

        default:
          return node.children ? processInlines(node.children, currentStyles) : [];
      }
    });
  };

  /**
   * ブロック要素の解析
   * 引用 (blockquote) を表 (Table) で実装し、多重線を再現
   */
  const transformBlocks = (
    nodes: any[], 
    context: { level: number; ordered: boolean } = { level: 0, ordered: false }
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
            spacing: { before: 240, after: 120 }
          }));
          break;

        case 'paragraph':
          results.push(new Paragraph({
            children: processInlines(node.children),
            spacing: { after: 120 }
          }));
          break;

        case 'blockquote': {
          // 引用の中身を再帰的に取得
          const quoteContent = transformBlocks(node.children, context);
          
          // 表を使って垂直線（左境界線）を作成
          results.push(new Table({
            width: { size: 100, type: WidthType.PERCENTAGE },
            borders: {
              top: { style: BorderStyle.NONE },
              bottom: { style: BorderStyle.NONE },
              left: { style: BorderStyle.SINGLE, size: 24, color: "C0C0C0" }, // 垂直線
              right: { style: BorderStyle.NONE },
              insideHorizontal: { style: BorderStyle.NONE },
              insideVertical: { style: BorderStyle.NONE },
            },
            rows: [
              new TableRow({
                children: [
                  new TableCell({
                    children: quoteContent as any,
                    margins: { left: 240, top: 100, bottom: 100 },
                    borders: {
                      top: { style: BorderStyle.NONE },
                      bottom: { style: BorderStyle.NONE },
                      left: { style: BorderStyle.NONE },
                      right: { style: BorderStyle.NONE },
                    }
                  })
                ]
              })
            ],
            // 前後の段落との間隔
            spacing: { before: 100, after: 100 }
          }));
          break;
        }

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
            indent: { left: (context.level * 360) + 360, hanging: 360 }
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
      alert('変換エラーが発生しました。コンソールを確認してください。');
    }
  };

  return <button onClick={handleDownload}>DOCXを保存</button>;
};

```

---

### 実装のポイント解説

* **多重の垂直線（Table Nesting）**:
引用ノード（`blockquote`）に遭遇するたびに、左側にだけ灰色の境界線を持たせた `Table` を作成し、その中にコンテンツを配置します。引用がネストしている場合、表の中にさらに表が作られるため、Word上で画像のように複数の垂直線が綺麗に並びます。
* **改行の完全保持**:
テキストを `\n` で分割し、2行目以降に `break: 1` プロパティを付与した `TextRun` を生成します。これにより、引用内の複数行テキストが1行にまとまってしまう現象を回避しています。
* **構造的なインデント**:
表（Table）の `TableCell` にマージン（`margins: { left: 240 }`）を設定しているため、垂直線とテキストの間に適切な余白が生まれます。

Windows + VSCodeの環境で、AIが回答した複雑なMarkdownをそのままDOCX化するツールとして、かなり完成度が高まったのではないでしょうか。

この変換ロジックをベースに、次は**ヘッダーやフッター、ページ番号の追加**なども検討されますか？あるいは、特定の**フォントファミリー（游明朝など）の指定**が必要であれば、すぐに対応可能です。
