// _document.tsx の Head 内に挿入
<script dangerouslySetInnerHTML={{ __html: `
  // WebSocketの生成を乗っ取り、HMR用の時だけ「切断されない偽物」を返す
  const OriginalWS = window.WebSocket;
  window.WebSocket = function(url, proc) {
    if (url.includes('hmr')) {
      return { 
        readyState: 1, close: () => {}, send: () => {}, 
        addEventListener: () => {}, removeEventListener: () => {} 
      };
    }
    return new OriginalWS(url, proc);
  };
`}} />
