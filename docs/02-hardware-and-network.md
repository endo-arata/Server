# 02. ハードウェアとネットワーク

## 2.1 ノード一覧

| ノード名 | 機種 | 主担当(提案) | 備考 |
|---|---|---|---|
| `studio-a` | Mac Studio M3 Ultra 512GB / 8TB | **Core**: 制御プレーン・データプレーン・ゲートウェイ + クラスタ推論シャード #0 | 常時稼働。単独障害でシステム全体が止まらないよう最も保守的に運用 |
| `studio-b` | 同上 | **Fast**: 中型モデル(高速会話/コーディング) + クラスタシャード #1 | |
| `studio-c` | 同上 | **Sense**: マルチモーダル(画像/音声/動画/OCR/埋め込み) + クラスタシャード #2 | |
| `studio-d` | 同上 | **Lab**: 実験・ファインチューニング・新モデル評価 + クラスタシャード #3 | 壊しても良い枠。クラスタ縮退時は最初に切り離す |

| `edge` | 小型 Linux サーバー(新規、D-022) | **Edge**: ルーター/FW、Home Assistant、カメラ AI、DNS、Tailscale Subnet Router | Mac 群が全停止しても「家」は動く常時稼働基盤 |

> Mac 4台は同一スペックなので役割は論理的なもの。役割は設定で入替可能にする。`edge` だけは物理的に別の役割。

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

回線は **NURO光 10Gbps**(D-014)。NURO の ONU 一体型ルーター(10GbE ポート付き)が WAN 側の起点になる。

```
[NURO ONU/ルーター] ─10GbE─ [edge: Proxmox → OPNsense (VLAN・FW・DNS・VPN) / HAOS / Frigate]
                                   │ 10GbE
                            [10GbE マネージドスイッチ]
                    ┌──────┬──────┼──────┬──────┬──────┬──────┐
                studio-a studio-b studio-c studio-d  NAS   PoE SW  Wi-Fi AP ×1〜2
                                                        (カメラ・音声端末)
```

| 項目 | 提案 | 備考 |
|---|---|---|
| WAN | NURO ONU をブリッジ相当(or DMZ)にして自前ルーターに全部渡す | NURO 機器は VLAN/細かい FW が組めないため二重ルーターを避ける(→OQ-004: 機種確認) |
| ルーター/FW | `edge` 上の OPNsense(D-022, 2.7 参照) | VLAN 間 FW、内部 DNS、DHCP、IDS。Tailscale Subnet Router も `edge` |
| ノード NIC | 内蔵 10GbE × 4台 | Mac Studio 標準 |
| スイッチ | 10GbE 8ポート以上(マネージド、SFP+/RJ45) | VLAN: `srv`(4台+NAS) / `client`(PC・スマホ) / `iot`(家電・カメラ) / `guest` |
| Wi-Fi | Wi-Fi 6E/7 AP × 1〜2(3〜4 部屋: D-023)を VLAN 対応で | IoT は 2.4GHz 隔離 SSID |
| PoE | 8 ポート PoE+ スイッチ(2.5GbE 可) | カメラ・音声端末・AP に給電 |
| 名前解決 | 自前ルーター or `studio-a` の内部 DNS(`*.home.arpa`) | Tailscale MagicDNS と併用 |
| 帯域の使い分け | TB5 メッシュ = 推論集合通信 / 10GbE = モデル配布・DB・バックアップ・UI | 10GbE で 600GB 配布は約 10 分 |

## 2.4 ストレージ設計(v0.2: 内蔵 8TB × 4 を前提)

内蔵 SSD は **8TB × 4 = 32TB**。1T 級 4bit(約 600GB)を 4 分割すると 1 ノードあたり約 160GB なので、
Titan 世代を複数保持しても内蔵で十分。NAS の役割は「倉庫」から「バックアップ・共有・メディア原本」へ縮小する。

| 層 | 用途 | 配置 | 容量目安 |
|---|---|---|---|
| Hot | 起動中モデル・KV キャッシュ・Postgres・作業領域 | 各ノード内蔵 8TB | ノードあたり 2〜3TB 使用 |
| Warm | ロード候補モデル(Titan 3 世代分のシャード、Fast/Sense 各数世代) | 各ノード内蔵 8TB の残り | ノードあたり 3〜4TB |
| Shared | 4 台で共有したいもの(モデル配布元、データセット、生成物) | `studio-a` の SMB 共有 or NAS | 数 TB |
| Cold/Backup | Postgres/オブジェクトのバックアップ、写真・動画原本、Time Machine | NAS(→OQ-002改: 新設 or 既存確認) | 20〜40TB |

### モデル配布の方針

- Hugging Face からの取得は `studio-a` が代表で行い、ハッシュ検証後に他ノードへ TB5/10GbE で配布(4 台が同じ 600GB を別々に落とさない)
- 各ノードは自分のシャードのみ Hot に置く。フルモデルの正本は `studio-a` の Warm に 1 部

### NAS の要否(再整理)

| 目的 | NAS なし | NAS あり |
|---|---|---|
| バックアップ | `studio-d` の内蔵 8TB を疑似バックアップ先に使う(同一筐体群なのでリスク残) | RAID + 別筐体で本来のバックアップになる |
| 写真・動画原本 | 内蔵に置く(32TB あれば当面十分) | 大容量・長期保存向き |
| 家族共有 | 不要(個人利用) | あれば便利 |

> **暫定結論**: Phase 1 では NAS なしで開始可能。Phase 4(記憶)までに、バックアップ専用の
> 小型 NAS(2〜4 ベイ、10GbE、20TB 級)を追加するのが安全。オフサイト(OQ-012)と合わせて決める。

## 2.5 電源・冷却・設置

- 実測報告: 4台フル負荷で **450〜800W**。家庭用コンセント1系統で賄える
- UPS: 1500VA クラス以上を推奨。停電時に Core を安全にシャットダウンし、NAS も保護
- 熱: Mac Studio は静音だが4台密集は避け、前面吸気/背面排気を確保
- 物理: 専用ラック棚 or デスクサイド。ケーブル長を短くするため 2×2 配置

## 2.7 `edge` ノード仕様(提案)

| 項目 | 提案 | 備考 |
|---|---|---|
| 筐体 | ファンレス or 静音 Mini PC(Intel N305/N355 or Ryzen 7000U 級)、**10GbE × 2 以上**(SFP+ or RJ45)+ 2.5GbE 数ポート | NURO 10G を活かすため WAN/LAN とも 10GbE 必須 |
| メモリ/SSD | 32〜64GB / NVMe 1〜2TB | Frigate の録画は別途 HDD/NAS |
| ハイパーバイザ | Proxmox VE | VM/LXC でルーターと HA を分離 |
| VM1: ルーター/FW | OPNsense(NIC を PCI パススルー) | VLAN、FW、DHCP、Unbound DNS、WireGuard(Tailscale 代替)、IDS(Suricata) |
| VM2: Home Assistant | Home Assistant OS | Zigbee/Thread ドングルを USB パススルー |
| LXC/VM3: 周辺 | Frigate(カメラ)、Mosquitto、Music Assistant、Tailscale Subnet Router、Uptime Kuma | GPU 無しでも Frigate は CPU/OpenVINO で可(Coral TPU 追加可) |
| 電源 | UPS 配下 | 停電時も FW/HA は最後まで生かす |
| 予算目安 | 8〜15 万円(本体)+ ドングル類 1 万円 | |

> なぜ Mac 上に置かないか: (1) macOS の再起動・OS 更新で家が止まる、(2) NIC パススルーや USB ドングル運用が macOS では困難、(3) 24/365 の常時稼働基盤と実験基盤を物理的に分けるのが安全。

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
