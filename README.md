ご指摘のケースは、Markdownの解析において「リストアイテムの1行目（親）」と「その後に続くコンテンツ（子）」の境界判断が非常にシビアな箇所です。

特に、ご提示いただいたような**「不揃いなインデント」や「太字記法内のスペース（`** プロ... **`）」**のようなイレギュラーな記述でも、親要素のテキストを絶対に逃さず、かつリンクや引用を正確に再現するようにロジックを再構築しました。

### 修正版：あらゆるMarkdown構造をDOCXへ変換するコード

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
   * インライン要素（テキスト、装飾、リンク）を解析
   * スタイル情報を再帰的に引き継ぐように修正
   */
  const processInlines = (nodes: any[], styles: { bold?: boolean; italics?: boolean } = {}): any[] => {
    return nodes.flatMap(node => {
      const currentStyles = {
        bold: styles.bold || node.type === 'strong',
        italics: styles.italics || node.type === 'emphasis'
      };

      switch (node.type) {
        case 'text':
          return new TextRun({ text: node.value, ...currentStyles });
        
        case 'strong':
        case 'emphasis':
          return processInlines(node.children, currentStyles);

        case 'link':
          // リンクタイトルの装飾（太字など）も維持しつつ、青色・下線付きでリンク化
          return new ExternalHyperlink({
            children: processInlines(node.children, currentStyles).map(run => {
              if (run instanceof TextRun) {
                return new TextRun({ ...run, color: '0563C1', underline: {} });
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
   * ブロック要素を解析
   * リストの親子関係と引用の深さを厳密に処理
   */
  const transformBlocks = (
    nodes: any[], 
    context: { level: number; ordered: boolean; quoteDepth: number } = { level: 0, ordered: false, quoteDepth: 0 }
  ): any[] => {
    const results: any[] = [];

    nodes.forEach(node => {
      switch (node.type) {
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
              left: { color: 'cccccc', space: 1, style: BorderStyle.SINGLE, size: 12 } 
            } : undefined
          }));
          break;

        case 'blockquote':
          // 2重引用 (>> ) の場合は、この再帰で quoteDepth が増え、インデントが深くなる
          results.push(...transformBlocks(node.children, { ...context, quoteDepth: context.quoteDepth + 1 }));
          break;

        case 'list':
          results.push(...transformBlocks(node.children, { ...context, ordered: node.ordered }));
          break;

        case 'listItem': {
          // 【超重要】ListItem直下の「テキスト(Paragraph等)」と「入れ子のList」を分離抽出
          const immediateContentNodes: any[] = [];
          const nestedListNodes: any[] = [];
          
          node.children.forEach((child: any) => {
            if (child.type === 'list') {
              nestedListNodes.push(child);
            } else if (child.type === 'paragraph') {
              // Paragraphの中身（textやstrong）を直接取り出す
              immediateContentNodes.push(...child.children);
            } else {
              immediateContentNodes.push(child);
            }
          });

          const lineChildren: any[] = [];
          
          // チェックボックス判定
          if (node.checked !== null && node.checked !== undefined) {
            lineChildren.push(new TextRun({ text: node.checked ? '☑ ' : '☐ ', font: 'MS Gothic' }));
          }

          // 親項目のテキストを生成（これが消えていた部分の修正）
          lineChildren.push(...processInlines(immediateContentNodes));

          results.push(new Paragraph({
            children: lineChildren,
            bullet: context.ordered ? undefined : { level: context.level },
            numbering: context.ordered ? { reference: 'main-numbering', level: context.level } : undefined,
            spacing: { after: 80 },
            // 引用内のリスト対応
            indent: context.quoteDepth > 0 ? { left: 720 * context.quoteDepth, hanging: 360 } : undefined
          }));

          // 子リストがあれば、レベルを1つ上げて再帰処理
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
      saveAs(blob, 'markdown-export.docx');
    } catch (e) {
      console.error(e);
      alert('DOCX変換中に予期せぬエラーが発生しました');
    }
  };

  return <button onClick={handleDownload}>DOCXをダウンロード</button>;
};

```

---

### 今回の修正のポイント

1. **親リスト項目の消失を防ぐ (ListItemの解析ロジック刷新)**:
`node.children` をスキャンし、「テキスト成分（Paragraph等）」と「子リスト（List）」を明確に分けました。以前は一括で再帰していたため、不揃いなインデントの際に親のテキストが「子リストの一部」と誤認され、表示がスキップされることがありました。これを分離したことで、`1. ** プログラミング言語 **` のような親行が確実に Paragraph として生成されます。
2. **2重引用 (>> ) への完全対応**:
`quoteDepth` を引数で管理するようにしました。引用ノードに入るたびに深さが加算され、Word側ではインデント（左余白）が `720 * quoteDepth` ずつ深くなります。これにより、階層に応じた視覚的な引用が再現されます。
3. **リンクの視認性向上**:
`ExternalHyperlink` 内のテキストに対して、強制的に青色 (`0563C1`) と下線 (`underline`) を付与しました。これで、Word上でも一目でリンクだと分かるようになります。
4. **記法内のスペースへの耐性**:
`processInlines` において、`strong` や `emphasis` の中身を再帰的に `processInlines` に渡すことで、`** プログラミング言語 **` のようにスペースが含まれていても、解析器が「テキスト」として返した値を漏らさず拾い上げるようにしました。

これで、かなり「意地悪な」マークダウンでも耐えられる堅牢なコードになったはずです。
実際に動かしてみて、特定の記法でまだ崩れる箇所があれば、そのマークダウンのサンプルと一緒にまた教えてください。

次はこのコードを既存のReactコンポーネントに組み込むお手伝いをしましょうか？それとも、docxの余白設定（上下左右のマージンなど）などのレイアウト調整に進みますか？
