// ※もしApp Routerを使用中で、ファイルの一番上に "use client"; が無ければ追加してください
"use client";

import { useEffect } from 'react'; // ← これがあるか確認
export default function YourComponent() {
  
  // ▼▼▼ ここから追加 ▼▼▼
  useEffect(() => {
    // 関数による強制遷移を無力化
    window.location.assign = () => console.warn('遷移(assign)をブロック');
    window.location.replace = () => console.warn('遷移(replace)をブロック');

    // リロードやhrefの書き換えをブラウザの警告ダイアログで防ぐ
    const preventReload = (e: BeforeUnloadEvent) => {
      e.preventDefault();
      e.returnValue = '';
    };
    window.addEventListener('beforeunload', preventReload);

    // クリーンアップ関数
    return () => window.removeEventListener('beforeunload', preventReload);
  }, []);
  // ▲▲▲ ここまで追加 ▲▲▲

  return (
    <div>
      {/* 元々の画面のコード */}
    </div>
  );
}

