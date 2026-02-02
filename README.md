度重なる修正にお付き合いいただきありがとうございます。ご指摘の現象は、MarkdownのAST（抽象構文木）における「リストアイテム内のネスト」と「引用の再帰構造」の解釈のズレが原因でした。

特に**サブ項目の重複**は、親のリストアイテムが子リストまで「自分のテキスト」としてスキャンしてしまっていたために発生していました。これを完全に分離し、**2重引用（>>）やリンク**もWordの形式に正しくマッピングした決定版のコードを作成しました。

### 修正版：Markdown to DOCX 変換ロジック

```tsx
import React from 'react';
import { 
  Document, 
  Packer, 
  Paragraph, 
  TextRun, 
  HeadingLevel, 
  ExternalHyperlink, 
  Table, TableRow, TableCell, 
  BorderStyle, WidthType,
  AlignmentType
} from 'docx';
import { saveAs } from 'file-saver';
import { unified } from 'unified';
import remarkParse from 'remark-parse';
import remarkGfm from 'remark-gfm';

export const MarkdownToDocx: React.FC<{ markdown: string }> = ({ markdown }) => {

  /**
   * インライン要素の解析
   * リンクや太字、チェックボックスなどを TextRun / ExternalHyperlink に変換
   */
  const processInlines = (nodes: any[]): any[] => {
    return nodes.flatMap(node => {
      switch (node.type) {
        case 'text':
          return new TextRun({ text: node.value });
        case 'strong':
          return processInlines(node.children).map(run => {
            if (run instanceof TextRun) return new TextRun({ ...run, bold: true });
            return run;
          });
        case 'emphasis':
          return processInlines(node.children).map(run => {
            if (run instanceof TextRun) return new TextRun({ ...run, italics: true });
            return run;
          });
        case 'link':
          // リンクタイトルのテキストを抽出し、ExternalHyperlinkを生成
          return new ExternalHyperlink({
            children: processInlines(node.children),
            link: node.url,
          });
        case 'inlineCode':
          return new TextRun({ 
            text: node.value, 
            font: 'Courier New', 
            shading: { fill: 'EEEEEE' },
            color: 'CC0000'
          });
        case 'break':
          return new TextRun({ break: 1 });
        default:
          return node.children ? processInlines(node.children) : [];
      }
    });
  };

  /**
   * ブロック要素の解析
   * 重複防止のため、リストアイテムとネストされたリストを厳密に分離して処理
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
            spacing: { before: 240, after: 120 }
          }));
          break;

        case 'paragraph':
          results.push(new Paragraph({
            children: processInlines(node.children),
            spacing: { after: 120 },
            // 引用の深さに応じてインデントと左線を付与（2重引用 >> に対応）
            indent: context.quoteDepth > 0 ? { left: 720 * context.quoteDepth } : undefined,
            border: context.quoteDepth > 0 ? { 
              left: { color: 'cccccc', space: 1, style: BorderStyle.SINGLE, size: 12 * context.quoteDepth } 
            } : undefined
          }));
          break;

        case 'blockquote':
          // 引用の深さをインクリメントして再帰処理
          results.push(...transformBlocks(node.children, { ...context, quoteDepth: context.quoteDepth + 1 }));
          break;

        case 'list':
          // リストそのものはコンテナなので、中身のlistItemに階層情報を渡す
          results.push(...transformBlocks(node.children, { 
            ...context,
            level: context.level, // ネストの深さはlistItem側の再帰で制御
            ordered: node.ordered 
          }));
          break;

        case 'listItem': {
          // 【修正ポイント】リストアイテム直下の「テキスト/段落」と「ネストされたリスト」を分離
          const immediateContent = node.children.filter((c: any) => c.type !== 'list');
          const nestedLists = node.children.filter((c: any) => c.type === 'list');

          const lineChildren: any[] = [];
          
          // チェックボックスの処理
          if (node.checked !== null && node.checked !== undefined) {
            lineChildren.push(new TextRun({ text: node.checked ? '☑ ' : '☐ ', font: 'MS Gothic' }));
          }

          // 段落の中身だけを抽出してListItemの1行目を作る
          const textContent = immediateContent.flatMap((c: any) => c.type === 'paragraph' ? c.children : [c]);
          lineChildren.push(...processInlines(textContent));

          results.push(new Paragraph({
            children: lineChildren,
            bullet: context.ordered ? undefined : { level: context.level },
            numbering: context.ordered ? { reference: 'main-numbering', level: context.level } : undefined,
            spacing: { after: 80 }
          }));

          // 【修正ポイント】ネストされたサブリストがある場合のみ、インデントレベルを上げて再帰
          if (nestedLists.length > 0) {
            results.push(...transformBlocks(nestedLists, { ...context, level: context.level + 1 }));
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
                verticalAlign: AlignmentType.CENTER
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
          config: [
            {
              reference: 'main-numbering',
              levels: Array.from({ length: 9 }, (_, i) => ({
                level: i,
                format: 'decimal',
                text: `%${i + 1}.`,
                alignment: AlignmentType.LEFT,
                style: { paragraph: { indent: { left: 720 * (i + 1), hanging: 360 } } }
              }))
            }
          ]
        },
        sections: [{ children: docElements }]
      });

      const blob = await Packer.toBlob(doc);
      saveAs(blob, 'formatted-report.docx');
    } catch (err) {
      console.error(err);
      alert('DOCX生成中にエラーが発生しました');
    }
  };

  return (
    <button onClick={handleDownload} className="px-4 py-2 bg-blue-500 text-white rounded">
      DOCXで保存
    </button>
  );
};

```

---

### 今回の見直し・修正点

* **サブ項目の重複解消**:
以前のコードでは `listItem` 内の `node.children` をすべて処理していたため、その中に含まれる「サブリスト（子ノード）」のテキストまで親の行に表示されていました。今回、`immediateContent`（テキスト部分）と `nestedLists`（サブリスト部分）を明示的にフィルタリングして分離したことで、重複を排除しました。
* **2重引用（>>）への対応**:
`quoteDepth` という変数を導入しました。引用がネストするたびにこの数値が増え、Word上でもインデント（`720 * depth`）と、左側の境界線の太さが強調されるようになり、視覚的に階層がわかるようになっています。
* **リンクの完全対応**:
`processInlines` 内で `link` 型のノードを見つけた際、`ExternalHyperlink` コンポーネントを使用するようにしました。これにより、Wordを開いたときに `[リンクタイトル](URL)` が青文字の下線付きリンクとして機能します。
* **チェックボックス**:
`remark-gfm` の `checked` プロパティを使い、`☑` または `☐` を行頭に挿入します。Wordの箇条書き機能と競合しないよう、記号として挿入しています。

これで、かなり複雑なMarkdownドキュメントでも、意図した通りの構造でDOCXに出力されるはずです。このコードをベースに、S3へのアップロード機能などと連携させてみてください。

他にも「ここをもっとこうしたい！」というスタイル上の要望（フォントをメイリオにしたい、表の枠線を消したい等）はありますか？
