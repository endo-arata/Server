# 04. 推論基盤とモデル戦略

## 4.1 クラスタ運用モード(提案: 「常時メッシュ + リクエスト単位ルーティング」)

D-009 の「動的切替」は**クラスタの組み替え**ではなく、**リクエストごとの振り分け**で実現する。
理由: 600GB 級モデルのロードに数分かかるため、組み替え型は待ち時間が大きい。
4台ともメモリに余裕(512GB)があるので、Titan シャードを常駐させたまま各ノードに固有モデルも常駐できる。

| モード | 状態 | 用途 |
|---|---|---|
| **Titan** | 4ノード テンソル並列(MLX+JACCL/RDMA) で 1T 級 MoE を常駐 | 難問・長文推論・重要判断・研究 |
| **Fast** | `studio-b` 単体で 27B〜120B 級を常駐 | 会話・コーディング・要約・ルーティング判断 |
| **Sense** | `studio-c` 単体でVLM/STT/TTS/OCR/埋め込み | 画像・音声・動画・文書取り込み |
| **Lab** | `studio-d` 単体、任意モデル | 評価・LoRA・新モデル試用 |
| **Degraded** | 1台欠けたとき: Titan 停止、残り3台は単体モード継続。`studio-a` 単体で 671B 4bit を代替起動(約21 tok/s) | 障害時 |

### GPU 競合の扱い

Titan が生成中は全ノードの GPU が使われるため、Fast/Sense の応答が遅くなる。
個人利用なので**優先度付きキュー**で解決する:

1. 対話中のユーザー要求(Fast/Sense)を最優先
2. Titan の長時間ジョブは中断可能(トークン単位でプリエンプト)にする
3. バックグラウンドの自律タスクは夜間や空き時間にスケジュール

## 4.2 モデル階層(2026-09 時点の候補)

> モデル名は流動的。**評価ハーネス(4.4)で入替を判定**し、ここは定期更新する。
> 以下は公開情報ベースの候補であり、構築時に再確認する。

| Tier | 役割 | 候補ファミリ | サイズ/量子化 | 期待速度 |
|---|---|---|---|---|
| T0 Titan | 最大知能 | Kimi K2/K3 系 (1T MoE, ~32B active), DeepSeek V3.x/V4 系 (671B MoE), Qwen3.x Max 級 | 4bit MLX, 4台分散 | 25〜32 tok/s |
| T1 Heavy | 単体で動く大型 | DeepSeek V3.1 671B, Qwen3 235B-A22B, GLM-5.x 系 | 4bit MLX, 1台 | 20〜32 tok/s |
| T2 Fast | 日常会話・コーディング | Qwen3.x 27B〜80B, GLM-5.2 (SWE 特化), Qwen3-Coder | 8bit/6bit MLX | 60〜130 tok/s |
| T3 Tiny | ルーティング・分類・下書き | 3B〜9B (Qwen3.x, Gemma, Llama 3.2) | 8bit | 150〜240 tok/s |
| E Embed | 埋め込み・リランク | bge-m3, Qwen3-Embedding/Reranker | fp16 | — |
| V Vision | 画像理解 | Qwen-VL 系, Pixtral, InternVL | 8bit mlx-vlm | — |
| A Audio | STT/TTS | Whisper large-v3-turbo (mlx-whisper), Kokoro/Fish-Speech/CosyVoice | — | リアルタイム |
| G Gen | 画像/動画生成 | FLUX 系, SDXL, (動画は Lab 枠で試用) | MLX / DiffusionKit | — |
| J JP | 日本語特化 | Swallow, ELYZA, PLaMo, Qwen 日本語 | Lab で評価し T2 と比較 | — |

## 4.3 ルーティング設計

```
要求 ─► [Router]
          │ 1. 明示指定 (model=…) があればそれ
          │ 2. T3 分類器が {難易度, モダリティ, 言語, ツール要否, 長さ} を判定
          │ 3. ポリシー表で Tier を決定
          │ 4. 対象ノードの負荷・キュー長でフォールバック
          ▼
   Titan / Heavy / Fast / Sense / (Cloud fallback: 明示許可時のみ)
```

| 判定例 | Tier |
|---|---|
| 「ざっくり要約して」「この文を直して」 | T2 |
| 「設計を比較して根拠付きで結論を」「数学の証明」「長文の矛盾検出」 | T0 |
| 画像添付・音声入力 | V/A → 結果を T2/T0 へ |
| コード生成・PR 作成 | T2(コーディング特化)、レビューは T0 |
| 深夜の自律リサーチ | T0(空いているため) |

ルーターの実装候補: LiteLLM(設定駆動)か、FastAPI で自作(分類器連携が柔軟)。**自作を推奨**(ルーティングログ自体をナレッジ化したいため)。

## 4.4 評価ハーネス(モデル入替の判断根拠)

- 個人用ベンチセット(日本語会話・要約・コーディング・推論・ツール呼び出し・長文)各 20〜50 問
- 指標: 正答率(LLM-as-judge は T0 が担当)、tok/s、TTFT、メモリ、電力
- 新モデルは `studio-d` で自動評価 → 閾値超えなら Tier 昇格を提案 → オーナー承認で切替
- 結果は Postgres に蓄積し Grafana で可視化

## 4.5 分散推論スタックの選定

| 候補 | 長所 | 短所 | 判断 |
|---|---|---|---|
| MLX distributed + JACCL | Apple 公式、RDMA 対応、テンソル並列、`mlx.launch` で運用が楽 | 自分で hostfile/起動を管理 | **主軸** |
| exo 1.0 | 自動発見・ゼロ設定、RDMA 対応、GUI あり | 単一ストリーム前提、同時要求で不安定報告 | 検証・比較用 |
| llama.cpp RPC | GGUF 互換 | TCP のみ、遅い | Titan には不採用。GGUF 資産の互換用 |

`mlx.distributed_config --hosts studio-a,studio-b,studio-c,studio-d --backend jaccl --auto-setup` で hostfile を生成し、
`mlx.launch` で `mlx_lm.server` を分散起動する構成を第一候補とする。

## 4.6 未決事項(→ 99 参照)

- クラウドフォールバック(Claude 等)を許可するか、条件は何か(OQ-003)
- 日本語性能の重み付け(OQ-006)
- 画像/動画生成をどこまで本気でやるか(OQ-007)
