# linkerd2-proxy

- [linkerd2-proxy](#linkerd2-proxy)
  - [Primary Directories and Their Functions](#primary-directories-and-their-functions)
  - [High-Level Processing Flow](#high-level-processing-flow)
  - [Sequence: Inbound (Receive) Flow](#sequence-inbound-receive-flow)
  - [Sequence: Outbound (Send) Flow](#sequence-outbound-send-flow)
  - [Sequence: Identity / Certificate Acquisition Flow](#sequence-identity--certificate-acquisition-flow)
  - [Reference Files (Excerpt)](#reference-files-excerpt)

## Primary Directories and Their Functions

- **`linkerd2-proxy/`** - 実行バイナリ（`src/main.rs`）とランタイムの起点。Proxy のエントリポイント。
- **`linkerd/`** - 多数の内部クレート群（`app`, `identity`, `inbound`, `outbound`, `dst` など）でプロキシのロジックを提供。
  - `linkerd/app` - アプリ全体の `Config` と `App::build` により、Identity、DST（Destination）、ポリシー、Inbound/Outbound スタックを組み立てる。
  - `linkerd/identity` - 証明書取得と mTLS の管理（SPIFFE / SPIRE 連携）。
  - `linkerd/inbound` - 受信パス（外部からの接続をローカルサービスへ転送）。
  - `linkerd/outbound` - 送信パス（ローカルアプリから外部サービスへルーティング）。
- **`spiffe-proto/`** - SPIFFE 関連のプロトバッファ定義など。
- **その他**: `metrics`, `tls`, `dns`, `router`, `policy` など、プロキシの機能ごとに分割。

## High-Level Processing Flow

- 起動 (main.rs)
  - `Config::try_from_env()` で設定を取得
  - `Config::build(...)` で `App` を構築（Identity, DST, Inbound, Outbound, Admin, Tap など）
  - 各リスナ（inbound/outbound/admin）を bind し、`serve` を spawn
- ランタイムは環境変数でコア数を調整（`rt.rs`）
- Identity が初期化されるまでプロキシは待機（`await_identity`）し、定期的に警告を出す

## Sequence: Inbound (Receive) Flow

```mermaid
sequenceDiagram
  %% インバウンド：外部 -> Proxy(inbound) -> ローカルサービス
  participant External as "外部クライアント"
  participant ProxyInbound as "Proxy (inbound)"
  participant Identity as "Identityサービス"
  participant Local as "ローカルサービス"

  External->>ProxyInbound: TCP/TLS 接続
  note right of ProxyInbound: TLS ハンドシェイク (mTLS の場合は相互認証)
  ProxyInbound->>Identity: 証明書検証/サーバ名確認
  Identity-->>ProxyInbound: 証明書検証結果
  ProxyInbound->>Local: ルーティング / ポリシー適用して転送
  Local-->>ProxyInbound: 応答
  ProxyInbound-->>External: 応答返却
```

**ポイント**: Inbound 側は接続を受けてから Identity（ローカルの SVID 層）による検証やポリシー照合を行い、ローカルソケットへ転送します。

## Sequence: Outbound (Send) Flow

```mermaid
sequenceDiagram
  %% アウトバウンド：ローカルアプリ -> Proxy(outbound) -> Destination -> リモートサービス
  participant App as "ローカルアプリ"
  participant ProxyOut as "Proxy (outbound)"
  participant DST as "Destinationコントロール (dst)"
  participant Remote as "リモートサービス"

  App->>ProxyOut: 接続/リクエスト
  ProxyOut->>DST: サービス名解決 / ポリシー確認
  DST-->>ProxyOut: エンドポイント + ポリシー
  note right of ProxyOut: ルーティング & LoadBalance
  ProxyOut->>Remote: 接続 (mTLS or plain) とリクエスト
  Remote-->>ProxyOut: レスポンス
  ProxyOut-->>App: レスポンス返却
```

**ポイント**: Outbound は DST（制御プレーン）からエンドポイントとプロファイルを受け取り、ポリシーやリトライなどを適用して送信します。

## Sequence: Identity / Certificate Acquisition Flow

```mermaid
sequenceDiagram
  participant Proxy as "Proxy"
  participant IdentitySvc as "Identity サービス (例: SPIRE)"
  participant CA as "CA / 証明書プロバイダ"

  Proxy->>IdentitySvc: CSR / SVID 取得要求
  IdentitySvc->>CA: 証明書署名要求
  CA-->>IdentitySvc: 署名済み証明書
  IdentitySvc-->>Proxy: 証明書 + 信頼チェーン
  note right of Proxy: 取得した証明書を使用して mTLS を開始
```

**補足**: `linkerd/identity` モジュールがこの役割を持ち、プロキシが起動してリスナを bind する前に有効な証明書を取得／検証します。

## Reference Files (Excerpt)

- `linkerd2-proxy/src/main.rs` — エントリと起動シーケンス
- `linkerd2-proxy/src/rt.rs` — ランタイム構築
- `linkerd/app/src/lib.rs` — `Config::build` と `App` 組立て
- `linkerd/identity` — 証明書 / mTLS まわり
- `linkerd/inbound` / `linkerd/outbound` — 各パスの実装
