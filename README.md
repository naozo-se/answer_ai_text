```
const { StorageBrowser } = createStorageBrowser({
  config: {
    // ...他の設定
    
    /**
     * ファイルのバリデーション関数
     * @param {File} file - アップロードされようとしているファイルオブジェクト
     * @returns {boolean | string} - trueなら許可、falseまたは文字列なら拒否
     */
    validateFile: (file) => {
      // --- 1. サイズ制限の設定 (例: 5MB) ---
      const MAX_SIZE_MB = 5;
      const MAX_SIZE_BYTES = MAX_SIZE_MB * 1024 * 1024;

      if (file.size > MAX_SIZE_BYTES) {
        // UI上でエラー理由を表示できる場合があるため、コンソールにも出すと親切です
        console.warn(`ファイルサイズが大きすぎます: ${file.name} (${(file.size / 1024 / 1024).toFixed(2)}MB)`);
        return false; // または "File size exceeds 5MB" などの文字列
      }

      // --- 2. 拡張子(形式)制限の設定 ---
      // 許可する拡張子のリスト（小文字で定義）
      const ALLOWED_EXTENSIONS = ['.png', '.jpg', '.jpeg', '.pdf'];
      
      // ファイル名から拡張子を取得して小文字化
      const fileName = file.name.toLowerCase();
      const isValidExtension = ALLOWED_EXTENSIONS.some(ext => fileName.endsWith(ext));
      
      // (補足) mimeTypeでチェックしたい場合は file.type を使います
      // const isValidMime = ['image/png', 'image/jpeg', 'application/pdf'].includes(file.type);

      if (!isValidExtension) {
        console.warn(`許可されていないファイル形式です: ${file.name}`);
        return false;
      }

      // すべてのチェックを通過したら許可
      return true;
    },
  },
});
```
