リンクが消えてしまう問題、何度も調整にお手間をかけさせてしまい申し訳ありません。

原因として最も可能性が高いのは、**`docx`ライブラリの `ExternalHyperlink` が「子要素として `TextRun` しか受け付けない」という極めて厳格な仕様**を持っていることです。

これまでのコードでは、再帰処理の中で `TextRun` 以外のオブジェクトが混入したり、`instanceof` による型チェックがライブラリの複数バージョン混在などで正しく機能せず、結果としてリンク全体が「不正な要素」としてWord側で無視（削除）されていた可能性があります。

リンクの中身を確実に「TextRunの配列」に固定して構築する、最も堅牢なロジックに差し替えました。

### 修正版：リンク表示を最優先した変換ロジック

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
   * リンク(link)内のテキストを確実にTextRunとして抽出する
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
          return [new TextRun({ text: node.value, ...currentStyles })];
        
        case 'strong':
        case 'emphasis':
        case 'delete':
          return processInlines(node.children, currentStyles);

        case 'link': {
          // 【最重要】リンク内のテキストに対して、リンク用のスタイルを直接適用して再帰取得
          const linkTextRuns = processInlines(node.children, {
            ...currentStyles,
            color: '0563C1', // 標準的なリンクの青色
            underline: { color: '0563C1' }
          });

          // docxのExternalHyperlinkは children に TextRunの配列を求めているため
          // 配列の中にさらに配列や別オブジェクトが入らないようにフラット化して返す
          return [new ExternalHyperlink({
            children: linkTextRuns.filter(child => child instanceof TextRun),
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
   * ブロック要素の解析（段落、リスト、引用、水平線、テーブル）
   */
  const transformBlocks = (
    nodes: any[], 
    context: { level: number; ordered: boolean; quoteDepth: number } = { level: 0, ordered: false, quoteDepth: 0 }
  ): any[] => {
    const results: any[] = [];

    nodes.forEach(node => {
      switch (node.type) {
        case 'thematicBreak': // 地平線 ---
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
            indent: context.quoteDepth > 0 ? { left: 720 * context.quoteDepth } : undefined,
            border: context.quoteDepth > 0 ? { 
              left: { color: 'cccccc', space: 1, style: BorderStyle.SINGLE, size: 24 } 
            } : undefined
          }));
          break;

        case 'blockquote': // 引用
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
          // Paragraphノードなどを介さず、直接インライン要素として抽出
          const flatInlines = immediateContentNodes.flatMap(c => c.type === 'paragraph' ? c.children : [c]);
          lineChildren.push(...processInlines(flatInlines));

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
      saveAs(blob, 'markdown-report.docx');
    } catch (e) {
      console.error('Docx Error:', e);
      alert('変換エラーが発生しました。コンソールを確認してください。');
    }
  };

  return <button onClick={handleDownload}>DOCXを保存</button>;
};

```

---

### 今回の「リンク復活」のための重要変更点

* **スタイルのカスケード適用**:
`processInlines` の第2引数に `color` や `underline` を追加しました。これにより、リンクノードを見つけた際、その中身（テキスト）を解析する再帰呼び出しに対して「お前は青くて下線付きだぞ」という情報を直接流し込み、確実に `TextRun` にそのスタイルが乗るようにしました。
* **ExternalHyperlink の子要素フィルタリング**:
`linkTextRuns.filter(child => child instanceof TextRun)` を追加しました。`ExternalHyperlink` は `Paragraph` とは異なり、子要素に他の複雑なオブジェクトが混ざることを許容しません。純粋な `TextRun` インスタンスのみを渡すことで、Wordが要素を拒絶して消去するのを防いでいます。
* **不揃いなListItem内のフラット化**:
不揃いなインデントの場合、`listItem` の子が `paragraph` だったり `text` だったりするため、`flatInlines` という処理を挟んで強制的に平滑化してから `processInlines` に渡すようにしました。

これで、`[リンクの例](URL)` の文字列が消えることなく、Word上で正しくリンクとして出力されるはずです。

もし、これでもまだ特定のリンクだけ消えるといった場合は、そのMarkdownの前後数行を教えていただけますか？特定の記号との組み合わせ（例：太字の中のリンクなど）が影響している可能性を潰せます。

このロジックでうまく出力できましたら、次はWordファイルの**余白設定やページサイズ**などのレイアウト調整をお手伝いしましょうか？
