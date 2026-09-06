# 02. ハードウェアとネットワーク

## 2.1 ノード一覧

| ノード名 | 機種 | 主担当(提案) | 備考 |
|---|---|---|---|
| `studio-a` | Mac Studio M3 Ultra 512GB | **Core**: 制御プレーン・データプレーン・ゲートウェイ + クラスタ推論シャード #0 | 常時稼働。単独障害でシステム全体が止まらないよう最も保守的に運用 |
| `studio-b` | 同上 | **Fast**: 中型モデル(高速会話/コーディング) + クラスタシャード #1 | |
| `studio-c` | 同上 | **Sense**: マルチモーダル(画像/音声/動画/OCR/埋め込み) + クラスタシャード #2 | |
| `studio-d` | 同上 | **Lab**: 実験・ファインチューニング・新モデル評価 + クラスタシャード #3 | 壊しても良い枠。クラスタ縮退時は最初に切り離す |

> 4台とも同一スペックなので役割は論理的なもの。役割は設定で入替可能にする。

### メモリ予算(1ノードあたり 512GB)

| 用途 | 予算 | 説明 |
|---|---|---|
| クラスタ推論シャード(Titan) | 〜200GB | 1T級 MoE 4bit(約600〜650GB)を4分割 → 約160GB + KVキャッシュ |
| ノード固有モデル | 〜200GB | Fast: 27B〜120B級, Sense: 画像/音声/埋め込み, Lab: 実験 |
| OS + サービス + ヘッドルーム | 〜100GB | macOS, コンテナ, DB, ページアウト回避 |

> Titan シャードを**常駐**させることで、600GB のロード時間(SSDから数分)を毎回払わずに済む。

## 2.2 Thunderbolt 5 メッシュ(推論バックプレーン)

- 4ノード **フルメッシュ**(各ノードが他3台と直結)。デイジーチェーンは不可
- 必要ケーブル: 6本(TB5 対応、短いほど良い。0.5〜1m)
- 各ノード 3ポート消費(M3 Ultra は TB5 × 6: 背面4 + 前面2)。残り3ポートを外付けストレージ等に使う
- **注意**: 報告によれば Ethernet ジャック隣接の TB5 ポートは RDMA リンクを張れない場合がある → 配線前に `rdma_ctl`/`ifconfig` で `rdma_enX` の出現を確認する
- 帯域: ポートあたり 80Gbps 対称。RDMA 時の往復レイテンシ 5〜50µs(TCP 時は約300µs)

### RDMA 有効化手順(各ノード1回)

1. macOS Tahoe 26.2 以降に更新
2. リカバリモードで起動(電源ボタン長押し)
3. ユーティリティ → ターミナル → `rdma_ctl enable`
4. 再起動
5. システム設定に RDMA over Thunderbolt の項目が出ることを確認

### 論理トポロジ図

```
        studio-a ───── studio-b
           │ ╲       ╱ │
           │   ╲   ╱   │      すべて TB5 直結 (6本)
           │   ╱   ╲   │
           │ ╱       ╲ │
        studio-c ───── studio-d
```

## 2.3 LAN(制御・サービス・ストレージ用)

TB5 メッシュは推論の集合通信専用にし、通常のトラフィックは Ethernet に分離する。

| 項目 | 提案 | 備考 |
|---|---|---|
| ノード NIC | 内蔵 10GbE × 4台 | Mac Studio 標準 |
| スイッチ | 10GbE 8ポート以上(マネージド) | VLAN で IoT / サーバー / クライアントを分離(→OQ-004) |
| ストレージ | NAS(10GbE, NVMe or HDD RAID) or 各ノード直付け TB5 NVMe | モデル倉庫・バックアップ・メディア(→OQ-002) |
| 名前解決 | 内部 DNS(`*.home.arpa` or 独自ドメインの内部ゾーン) | Tailscale MagicDNS と併用 |

## 2.4 ストレージ設計(暫定)

内蔵 2TB × 4 = 8TB は**モデルには足りない**(1T級 4bit だけで 600GB、複数世代を保持したい)。

| 層 | 用途 | 提案 |
|---|---|---|
| Hot(ノード内蔵 SSD) | 起動中モデル・KVキャッシュ・DB | 内蔵 2TB |
| Warm(ノード直付け) | ロード候補モデル・埋め込み索引 | TB5 NVMe エンクロージャ 4〜8TB を各ノードに(任意) |
| Cold(共有) | モデル倉庫・全バックアップ・メディア原本 | NAS 40TB+(HDD RAID6 + NVMe キャッシュ) |

> 最小構成では NAS 1台で Warm/Cold を兼ねる。10GbE 経由で 600GB のロードは約10分。

## 2.5 電源・冷却・設置

- 実測報告: 4台フル負荷で **450〜800W**。家庭用コンセント1系統で賄える
- UPS: 1500VA クラス以上を推奨。停電時に Core を安全にシャットダウンし、NAS も保護
- 熱: Mac Studio は静音だが4台密集は避け、前面吸気/背面排気を確保
- 物理: 専用ラック棚 or デスクサイド。ケーブル長を短くするため 2×2 配置

## 2.6 OS / コンテナランタイム提案

| 選択肢 | 評価 | 結論 |
|---|---|---|
| macOS ネイティブ + MLX + コンテナ(OrbStack/Colima) | GPU/NE をフル活用。MLX は macOS 前提。コンテナは Linux 依存サービス(Postgres, n8n 等)用 | **採用** |
| Asahi Linux | GPU/NE ドライバが不十分。MLX 非対応 | 不採用 |
| Proxmox / k3s | macOS 上では成立しない(VM は GPU 使えず) | 不採用 |
| Apple Container / VZ ベース Linux VM | Linux 専用ワークロードの隔離に部分利用 | 補助的に採用可 |

> 「サービスは Linux コンテナ、推論は macOS ネイティブ」の二層で構成する。
> 推論サーバーをコンテナ化しないのは、コンテナ内から Metal/RDMA に触れないため。

## 参考

- Apple WWDC26 Session 233 "Explore distributed inference and training with MLX"
- AppleInsider 2025-12-20 "AI calculations on Mac cluster gets a big boost from new RDMA support on Thunderbolt 5"
- Jeff Geerling "1.5 TB of VRAM on Mac Studio - RDMA over Thunderbolt 5"
- runaihome "Mac Studio Cluster for Trillion-Parameter AI in 2026"
- sean-weldon "Clustering Mac Studios for Local AI" (2026-06-16)
