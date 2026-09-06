# 10. ロードマップ

> 計画フェーズを最重視するため、**Phase 0 が完了(オーナーの打ち切り宣言)するまで実装に入らない。**

| Phase | 名前 | 内容 | 完了条件 |
|---|---|---|---|
| 0 | **設計** | 本文書群を質問→決定のループで詰める | オーナーが「計画打ち切り」を宣言。全 OQ が決定 or 意図的保留 |
| 1 | 土台 | RDMA 有効化、TB5 メッシュ配線、10GbE/VLAN、NAS、Tailscale、Ansible で 4台を宣言的管理 | 4台間で `mlx.distributed` の疎通テスト成功。外出先から Web に到達 |
| 2 | 単体推論 | `studio-b` に Fast、`studio-c` に Sense、`studio-a` に Router/Embed。Open WebUI で会話 | 日本語会話 60 tok/s 以上、画像/音声入力が動く |
| 3 | Titan | 4台テンソル並列で 1T 級を常駐。Router が難易度で振り分け | 25 tok/s 以上で安定 24h。Degraded 切替が自動 |
| 4 | 記憶 | Postgres/pgvector/グラフ、Profile、インタビュー、Topic Wiki の初期生成 | 「先週何を決めた?」に出典付きで答える |
| 5 | 入口 | Discord Bot、音声(HA Assist)、承認 UI、IDE 連携 | スマホから音声で家電を操作し、Discord で承認できる |
| 6 | 自律 | Morning Brief、Research Scout、Repo Guardian、Media Librarian、Sentinel | 1 週間無操作でナレッジと家が維持される |
| 7 | 拡張 | 7章のカタログ B 群、画像/動画生成、ファインチューニング、評価ハーネスの自動昇格 | 新モデルが自動評価され提案が届く |

## 各 Phase の見積り(構築時間の目安。設計打ち切り後に再見積り)

| Phase | 目安 |
|---|---|
| 1 | 2〜3 日(配線・NW・NAS の物理作業含む) |
| 2 | 2〜3 日 |
| 3 | 3〜5 日(RDMA の試行錯誤を含む) |
| 4 | 1〜2 週 |
| 5 | 1 週 |
| 6 | 2〜3 週 |
| 7 | 継続 |
