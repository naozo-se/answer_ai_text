失礼いたしました！Mermaid構文でのシーケンス図ですね。こちらの方が、ドキュメントなどにもそのまま貼り付けて使いやすいかと思います。

2つの方式における「会話が始まるまでの流れ」をMermaidで表現しました。

### 1. エフェメラルトークン方式
まずはクライアントが「身分証（トークン）」をゲットしてから、自分でAzureに接続しにいく方式です。

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

    Note over Client, Azure: 【会話フェーズ】
    Client<-->>Azure: WebRTC通信確立（音声・テキストの直接P2P通信）
```

---

### 2. SDPパススルー方式
クライアントには一切トークンを渡さず、バックエンドが「接続の仲介」だけを代理で行う方式です。

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
    
    Note right of Backend: バックエンド側で認証情報(APIキー等)や<br>設定をこっそり付与して転送する
    Backend->>Azure: SDP Offer をパススルー送信
    Azure-->>Backend: SDP Answer 返却
    Backend-->>Client: SDP Answer をそのまま転送

    Note over Client, Azure: 【会話フェーズ】
    Note right of Client: 接続先情報(SDP)が分かったので<br>直接Azureと通信を始める
    Client<-->>Azure: WebRTC通信確立（音声・テキストの直接P2P通信）
```

### ポイント
Mermaid図の最後のステップ（`WebRTC通信確立`）を見ていただくと分かりやすいのですが、**どちらの方式を通ったとしても、最終的な「会話の経路」は全く同じ（クライアントとAzureの直接通信）**になります。

違うのは「1〜4番のやり取り」だけです。設計のイメージを掴む手助けになれば幸いです！
