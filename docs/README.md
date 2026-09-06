# AI Home Server 計画書 (Project "Studio Quad")

4台の Mac Studio (M3 Ultra / 512GB Unified Memory / 8TB SSD) を核に、
ローカルLLM中心で「AIをふんだんに搭載した個人用サーバー基盤」を構築するための計画書群。

本プロジェクトは **計画フェーズを最重視** する。実装に着手する前に、ここにある文書を
質問→決定→反映のループで磨き続ける。

## 文書構成

| # | ファイル | 内容 | 状態 |
|---|---|---|---|
| 00 | [00-decision-log.md](00-decision-log.md) | これまでの決定事項(Q&Aの記録) | 更新中 |
| 01 | [01-vision-and-principles.md](01-vision-and-principles.md) | ビジョン・設計原則・非目標 | v0.1 |
| 02 | [02-hardware-and-network.md](02-hardware-and-network.md) | 4台の物理構成・Thunderbolt 5 メッシュ・RDMA・LAN・電源 | v0.2 |
| 03 | [03-platform-architecture.md](03-platform-architecture.md) | 5プレーン構成(制御/推論/データ/エージェント/インターフェース) | v0.1 |
| 04 | [04-inference-and-models.md](04-inference-and-models.md) | クラスタ運用モード・モデル階層・ルーティング | v0.1 |
| 05 | [05-knowledge-and-rag.md](05-knowledge-and-rag.md) | 「データが無い」前提の自律的ナレッジ構築とRAG・初期 Topic Wiki | v0.2 |
| 06 | [06-agents-and-automation.md](06-agents-and-automation.md) | エージェント実行基盤・自律オートメーション | v0.1 |
| 07 | [07-life-integrations.md](07-life-integrations.md) | 生活連携カタログ・スマートホーム構築計画・音声設計 | v0.2 |
| 08 | [08-security-and-remote-access.md](08-security-and-remote-access.md) | VPN・認証・秘密管理・ゼロトラスト | v0.1 |
| 09 | [09-operations.md](09-operations.md) | 監視・バックアップ・更新・障害対応 | v0.1 |
| 10 | [10-roadmap.md](10-roadmap.md) | フェーズ分割と受け入れ基準 | v0.1 |
| 99 | [99-open-questions.md](99-open-questions.md) | 未決事項(次の質問候補) | 更新中 |

## 計画の進め方

1. Claude が AskUserQuestion で論点を提示する
2. 回答を `00-decision-log.md` に決定として記録する
3. 影響する文書を更新し、バージョンを上げる
4. `99-open-questions.md` から次の論点を取り出す
5. オーナーが「打ち切り」と言うまで繰り返す

## 前提となる事実(2026-09 時点で確認済み)

- macOS Tahoe 26.2 (2025-12-12 リリース) で **RDMA over Thunderbolt 5** が有効化可能
- MLX の **JACCL** バックエンドが RDMA を利用したテンソル並列を提供
- 4台の M3 Ultra クラスタで 1T パラメータ級 MoE (Kimi K2 Thinking) が **約25〜28 tok/s**、DeepSeek V3.1 671B が **約32 tok/s**
- RDMA なしでは 4ノードが 2ノードより遅くなる(通信オーバーヘッド)。RDMA は必須
- 単一ノード(512GB)でも DeepSeek V3.1 671B 4bit 級が **約21 tok/s** で動作

参考: Apple WWDC26 Session 233 / Jeff Geerling / AppleInsider / runaihome / sean-weldon (詳細は各文書末尾)
