```
yarn add @mohtasham/md-to-docx
```

```
import { convertMarkdownToDocx, downloadDocx } from "@mohtasham/md-to-docx";

const handleDownload = async (markdownText: string) => {
  // MarkdownをDocx(Blob)に変換
  const blob = await convertMarkdownToDocx(markdownText);
  // ブラウザでダウンロード実行
  downloadDocx(blob, "ai-response.docx");
};
```
