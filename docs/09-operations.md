# 09. 運用

## 9.1 監視

| 対象 | 指標 | 手段 |
|---|---|---|
| ノード | CPU/GPU/NE 使用率、メモリ圧、温度、電力、SSD 寿命 | macmon / powermetrics → Prometheus exporter |
| クラスタ | RDMA リンク状態、集合通信レイテンシ、シャード生存 | 自作 exporter(`ifconfig rdma_en*`, ヘルスチェック) |
| 推論 | tok/s、TTFT、キュー長、失敗率、モデル別利用量 | Router のメトリクス |
| エージェント | 実行数、成功率、承認待ち、予算消費 | ランタイムのメトリクス |
| データ | DB サイズ、索引鮮度、バックアップ成否 | Postgres exporter + ジョブログ |
| 通知 | 異常は Discord へ。深刻度別にチャンネル分け | Alertmanager |

## 9.2 バックアップ

| 対象 | 頻度 | 先 | 方式 |
|---|---|---|---|
| Postgres | 毎時 WAL + 毎晩フル | NAS → クラウド(暗号化, D-027) | pgBackRest + rclone crypt |
| オブジェクト(原本) | 毎晩 | NAS(RAID)+ クラウド(暗号化, D-027) | rclone crypt → Backblaze B2 / Cloudflare R2 等。鍵はオーナーのみ保持 |
| 設定・IaC | 変更時 | Git(このリポジトリ) | Ansible/compose を宣言的に |
| モデル | なし(再取得可) | NAS に倉庫 | ハッシュ台帳のみ保持 |
| macOS ノード | 週次 | NAS | Time Machine(システム部分のみ) |

復旧目標(暫定): RPO 1時間 / RTO 半日(Core 再構築を Ansible で自動化)

## 9.3 更新

- macOS: 検証は `studio-d` → 問題なければ他3台。RDMA/MLX 互換を必ず確認してから
- モデル: 4.4 の評価ハーネス経由。ロールバック用に前世代を NAS に保持
- コンテナ: Renovate/Dependabot 相当で PR 作成 → Lab で起動テスト → 承認で適用
- 本リポジトリ: 全変更を PR で。Claude Code(ローカルモデル版含む)がレビュー

## 9.4 障害対応(ランブック候補)

| 事象 | 対応 |
|---|---|
| 1ノード停止 | Router が Degraded へ切替。`studio-a` で 671B 単体を起動。Discord に通知 |
| RDMA リンク断 | 該当ケーブル/ポートを特定 → TCP フォールバック(遅いが動く)→ 物理確認 |
| Core 停止 | UPS で安全停止 → 再起動 → Ansible で整合性確認 → DB 整合性チェック |
| ディスク逼迫 | Sentinel が KV キャッシュ・古いモデル・ログを掃除 |
| 暴走エージェント | Sentinel が予算超過で停止 → 原因トレースをレポート |

## 9.5 電力・コスト

- 4台アイドル 約 40〜60W、フル負荷 約 600〜800W と報告あり。月間電気代を Grafana に表示
- 夜間の重いバッチは電力単価の安い時間帯に寄せる(契約次第→OQ-013)
