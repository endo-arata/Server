# 03. プラットフォームアーキテクチャ

## 3.1 5プレーン構成

```
┌──────────────────────────────────────────────────────────────┐
│ Interface Plane   Web UI / Discord / 音声 / モバイル / API / CLI │
├──────────────────────────────────────────────────────────────┤
│ Agent Plane       エージェントランタイム・ワークフロー・スケジューラ・ツール(MCP) │
├──────────────────────────────────────────────────────────────┤
│ Inference Plane   Router → Titan(4台TP) / Fast / Sense / Embed  │
├──────────────────────────────────────────────────────────────┤
│ Data Plane        Postgres(+pgvector) / オブジェクト / ナレッジグラフ / 時系列 │
├──────────────────────────────────────────────────────────────┤
│ Control Plane     Gateway・認証・秘密・監視・ログ・ジョブ制御・ノード管理  │
└──────────────────────────────────────────────────────────────┘
```

## 3.2 各プレーンの構成要素(提案)

### Control Plane(`studio-a`)

| 要素 | 候補 | 役割 |
|---|---|---|
| Reverse Proxy / Gateway | Caddy or Traefik | TLS終端、ルーティング、レート制限 |
| ID/認証 | Tailscale ID + Authelia/Pocket-ID (OIDC) | SSO。VPN 越しでも二段階 |
| 秘密管理 | 1Password Connect or Infisical or sops+age | APIキー・トークンを平文で置かない |
| 監視 | Prometheus + Grafana + node/macmon exporter | GPU/メモリ/温度/電力 |
| ログ・トレース | OpenTelemetry Collector + Loki + Tempo | 推論・ツール実行の全トレース |
| ジョブ/キュー | Temporal or Postgres ベース(pg-boss/River) | 長時間タスク・再試行・スケジュール |
| ノード管理 | Ansible(macOS 対応) + launchd | 4台の構成を宣言的に |

### Inference Plane(全ノード)

| 要素 | 候補 | 役割 |
|---|---|---|
| 分散推論(Titan) | MLX distributed + JACCL(RDMA) / 代替: exo 1.0 | 1T 級 MoE をテンソル並列 |
| 単体推論(Fast/Lab) | mlx-lm server / LM Studio server / llama.cpp(GGUF互換用) | OpenAI 互換 API |
| マルチモーダル(Sense) | mlx-vlm(画像), whisper.cpp / mlx-whisper(音声), Kokoro/Fish TTS, Florence/PaddleOCR | |
| 埋め込み/リランク | mlx 埋め込み(bge-m3, Qwen3-Embedding 等) | RAG 用 |
| ルーター | LiteLLM or 自作(FastAPI) | モデル選択・フォールバック・予算管理・キャッシュ |

### Data Plane(`studio-a`、NAS)

| 要素 | 候補 | 役割 |
|---|---|---|
| RDB + ベクトル | PostgreSQL 17 + pgvector + pg_search | メモリ・会話・文書・埋め込み |
| ナレッジグラフ | Postgres 上の三つ組 or Neo4j/Memgraph | エンティティ・関係・時系列 |
| オブジェクト | MinIO(NAS 上) | 文書原本・画像・音声・モデル成果物 |
| 時系列 | TimescaleDB(Postgres 拡張) | センサー・健康・利用ログ |
| 全文検索 | Postgres FTS + 日本語トークナイザ(pg_bigm/PGroonga) | 日本語対応 |

### Agent Plane(`studio-a` 実行、推論は全ノード)

| 要素 | 候補 | 役割 |
|---|---|---|
| エージェントランタイム | 自作(Python/TypeScript) + Claude Agent SDK 互換設計 | ツール呼び出しループ・メモリ・権限 |
| ワークフロー | n8n or Temporal | イベント駆動・スケジュール |
| ツール接続 | MCP サーバー群(自作+OSS) | 家電・カレンダー・GitHub・NAS・ブラウザ |
| ブラウザ自動化 | Playwright(Chromium) | Web 収集・操作 |
| サンドボックス | Linux VM / コンテナ | エージェントのコード実行隔離 |

### Interface Plane

| 要素 | 候補 | 役割 |
|---|---|---|
| Web UI | Open WebUI or LibreChat | チャット・ファイル・画像 |
| チャットアプリ | **Discord Bot**(D-016)。専用サーバーにチャンネル分け(会話/通知/承認/アラート/ログ) | どこからでも指示・通知・承認。LINE/Telegram は将来の通知専用 |
| 音声 | Home Assistant Assist + ローカル STT/TTS + スマートスピーカー | 家の中はハンズフリー |
| モバイル | Tailscale 経由で Web UI / PWA | |
| CLI/IDE | OpenAI 互換 API を Claude Code / Cursor / Continue 等から | |

## 3.3 ノード配置の要約

| プレーン | studio-a (Core) | studio-b (Fast) | studio-c (Sense) | studio-d (Lab) |
|---|---|---|---|---|
| Control | ● 全部 | エージェント | エージェント | エージェント |
| Inference | Titan #0, Embed | Titan #1, Fast | Titan #2, Sense | Titan #3, 実験 |
| Data | ● Postgres 等 | — | — | — |
| Agent | ● ランタイム・n8n | サンドボックス | サンドボックス | サンドボックス |
| Interface | ● Gateway・UI・Bot | — | — | — |

> Core が単一障害点になるのは意図的。個人利用では「復旧手順が明確な単一点」の方が
> 分散合意(etcd 等)より運用が楽。Core の SSD/DB は毎晩 NAS へスナップショット。

## 3.4 内部 API 契約(方針)

- 推論はすべて **OpenAI 互換 API** に正規化(`/v1/chat/completions`, `/v1/embeddings`, `/v1/audio/*`)
- ツールは **MCP** に正規化。エージェントは MCP クライアントとして統一
- イベントは **CloudEvents 形式** で Postgres の `events` テーブルに集約し、n8n/Temporal が購読
- 全リクエストに `trace_id` を付与し、OpenTelemetry へ
