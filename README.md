申し訳ありません！完全に私のミスです。

ご提示いただいたエラー画像にある通り、Mermaidのシーケンス図において双方向を示すつもりで書いた `<-->>` という矢印の構文が、標準的なパーサーで「無効な形式（INVALID）」として弾かれてしまっています。

安全で確実な標準構文（`->>` と `-->>`）を使って、双方向のやり取りを表現するように修正しました。こちらで問題なくレンダリングされるはずです！

### 1. エフェメラルトークン方式（修正版）

```mermaid
sequenceDiagram
    autonumber
    participant Client as クライアント<br>(ブラウザ)
    participant Backend as バックエンド<br>(自社サーバー)
    participant Azure as Azure OpenAI<br>(Realtime API)

    Note over Client, Azure: 【トークン取得フェーズ】
    Client->>Backend: トークン要求
    Note right of Backend: サーバーのAPIキー等を使って<br>Azureに認証をかける
    Backend->>Azure: トークン生成リクエスト (/client_secrets)
    Azure-->>Backend: エフェメラルトークン発行
    Backend-->>Client: エフェメラルトークンを渡す

    Note over Client, Azure: 【SDP交換（ネゴシエーション）フェーズ】
    Client->>Client: SDP Offer 作成
    Note right of Client: 受け取ったトークンを使って<br>直接Azureに接続要求を出す
    Client->>Azure: SDP Offer 送信 (Headerにトークン付与)
    Azure-->>Client: SDP Answer 返却

    Note over Client, Azure: 【会話フェーズ（WebRTC確立）】
    Client->>Azure: 音声・テキスト送信 (P2P通信)
    Azure-->>Client: 音声・テキスト受信 (P2P通信)
```

---

### 2. SDPパススルー方式（修正版）

```mermaid
sequenceDiagram
    autonumber
    participant Client as クライアント<br>(ブラウザ)
    participant Backend as バックエンド<br>(自社サーバー)
    participant Azure as Azure OpenAI<br>(Realtime API)

    Note over Client, Azure: 【SDP交換（ネゴシエーション）フェーズ】
    Client->>Client: SDP Offer 作成
    Note right of Client: トークンは持っていないので<br>とりあえずバックエンドに送る
    Client->>Backend: SDP Offer 送信
    
    Note right of Backend: バックエンド側で認証情報や<br>設定をこっそり付与して転送する
    Backend->>Azure: SDP Offer をパススルー送信
    Azure-->>Backend: SDP Answer 返却
    Backend-->>Client: SDP Answer をそのまま転送

    Note over Client, Azure: 【会話フェーズ（WebRTC確立）】
    Note right of Client: 接続先情報(SDP)が分かったので<br>直接Azureと通信を始める
    Client->>Azure: 音声・テキスト送信 (P2P通信)
    Azure-->>Client: 音声・テキスト受信 (P2P通信)
```

お手数をおかけしました！プレビュー環境やドキュメントツールなどで、こちらのコードをコピペして再度お試しいただけますでしょうか？
