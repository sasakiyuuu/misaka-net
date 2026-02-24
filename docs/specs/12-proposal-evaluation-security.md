# 12 Proposal Evaluation Security 仕様

## 1. 目的と適用範囲
本仕様は proposal 評価における collusion/Sybil/bootstrap bias 耐性を定義する。
本仕様の対策は **MUST** 実装されなければならない。

## 2. 規範キーワード
- **MUST**: 準拠実装に必須
- **SHOULD**: 特段の理由がない限り必須
- **MAY**: 任意

## 3. 脅威モデル
- `Collusion`: 複数主体の談合評価
- `Sybil`: 偽装多重IDでの票操作
- `Bootstrap bias`: 初期参加者への過剰有利

## 4. 必須防御メカニズム

### 4.1 evaluator stake cap（必須）
- 各 evaluator の有効 stake は **MUST** `CAP_STAKE = 2%` で上限化。
- 上限超過 stake は **MUST** 評価式から除外。

### 4.2 quadratic weighting（必須）
- evaluator 重みは **MUST** `w_i = sqrt(stake_i_capped)` を使用。
- 線形重みは **MUST NOT** 使用。

### 4.3 reviewer sampling（必須）
- reviewer は **MUST** consensus epoch（`02-consensus.md` §5）ごとに再抽選。
- 抽選シードは **MUST** `validator_set_hash || checkpoint_seq` を含む。
- 同一入力で抽選結果は **MUST** 再現可能。

### 4.4 reputation decay（必須）
- reputation は **MUST** 時間減衰。
- 式: `rep_t = rep_0 * exp(-lambda * t)`
- `t` は **MUST** consensus epoch（`02-consensus.md` §5）単位で測る。
- 初期値: `lambda = 0.05 / epoch`

## 5. 評価スコア式
定義:
- `rating_i`: evaluator i が proposal に付与する評価値（範囲 `0.0..1.0`）
- `rep_i`: evaluator i の時点 reputation（`rep_t` で計算）
- `N`: 当該 epoch で有効な sampled evaluator 数
- `t`: epoch 単位の経過時間

- `score_i = rating_i * sqrt(stake_i_capped) * rep_i`
- `ProposalScore = (1/N) * sum(score_i)`

採択条件:
- `ProposalScore >= THRESHOLD`（初期値 `0.60`）

### 5.1 計算配置モデル（Off-chain 計算 / On-chain 検証）
- `pearson` / `cosine` / `D_KL` 等の統計計算は **MUST** off-chain evaluation worker で実行する。
- on-chain 実行系は **MUST NOT** 上記統計量を raw 入力から再計算する。
- on-chain 実行系は **MUST** `evaluation_commitment_v1` を検証し、`input_commitment` と `metrics_commitment` の一致のみを判定根拠に使う。
- `input_commitment = SHA3-256(MCS-1(EvaluationInputCanonical))` を **MUST** 使用する。
- `metrics_commitment = SHA3-256(MCS-1(EvaluationMetricsCanonical))` を **MUST** 使用する。
- `EvaluationInputCanonical` は **MUST** MCS-1 構造体として次順序で保持する。
  1. `checkpoint_seq: u64`
  2. `proposal_id: bytes[32]`
  3. `sampled_reviewers: list<bytes[32]>`（bytewise 昇順、重複なし）
  4. `ratings: list<RatingEntry>`（`reviewer_pubkey` 昇順）
  5. `weights: list<WeightEntry>`（`reviewer_pubkey` 昇順）
  6. `rep: list<ReputationEntry>`（`reviewer_pubkey` 昇順）
- `EvaluationMetricsCanonical` は **MUST** MCS-1 構造体として次順序で保持する。
  1. `pearson: list<PairMetricEntry>`（`reviewer_a || reviewer_b` 昇順）
  2. `cosine: list<PairMetricEntry>`（`reviewer_a || reviewer_b` 昇順）
  3. `d_kl: list<ClusterMetricEntry>`（`cluster_id` 昇順）
  4. `suspect_labels: list<SuspectLabelEntry>`（`subject_id` 昇順）
  5. `threshold_snapshot: bytes[32]`

## 6. 指標固定化回避（必須）
- 単一利害関係者カテゴリ（組織・地域・資本属性）が合計重み `> 35%` となる状態を **MUST NOT** 継続許容。
- 連続 `3` epoch で超過した場合、システムは **MUST** 補正モードへ移行。

補正モード要件:
1. 超過カテゴリの重みを減衰
2. 非超過カテゴリのサンプリング比率を増加
3. 補正履歴を監査ログへ記録

## 7. 不正検知

### 7.1 Collusion 指標（必須）
定義:
- `v_i`: evaluator i の rating ベクトル（proposal 次元）
- `pearson(i,j)`: `corr(v_i, v_j)`
- `cosine(i,j)`: `(v_i · v_j) / (||v_i|| * ||v_j||)`

計算配置:
- `pearson(i,j)` と `cosine(i,j)` の算出は **MUST** off-chain evaluation worker で実行する。
- on-chain 実行系は **MUST** `evaluation_commitment_v1.metrics_commitment` と監査ログ上の `input_commitment` 一致のみを検証する。

閾値:
- `pearson(i,j) >= 0.90` かつ `cosine(i,j) >= 0.95` が `3` epoch 連続した evaluator 群は **MUST** collusion suspect と判定。

### 7.2 Sybil 指標（必須）
定義:
- `P_cluster`: suspect cluster の rating 分布
- `P_global`: 全 evaluator の rating 分布
- `D_KL(P_cluster || P_global)`: KL divergence
- `ONBOARDING_WINDOW_EPOCHS = 7`（初期値）

計算配置:
- `D_KL(P_cluster || P_global)` の算出は **MUST** off-chain evaluation worker で実行する。
- on-chain 実行系は **MUST NOT** 分布再構築による KL 再計算を行わず、`metrics_commitment` と `proof_bytes` の検証結果のみを採用する。

閾値:
- `D_KL(P_cluster || P_global) <= 0.05` かつ直近 `ONBOARDING_WINDOW_EPOCHS` epoch 内に ID 増加率が `>= 2x` の場合、当該 cluster は **MUST** sybil suspect と判定。

### 7.3 対処規則
- collusion/sybil suspect evaluator は **MUST** 1 epoch の隔離を適用し、評価重みを `0` に設定。
- 連続再発時、`rep_i` は **MUST** 追加で `0.5` 倍減衰。
- 検知/解除ルールは **MUST** `sys/params/*` に保存し、ガバナンス変更のみ許可。

### 7.4 `evaluation_commitment_v1` と検証規則（必須）
`RatingEntry`（MCS-1 順）:
1. `reviewer_pubkey: bytes[32]`
2. `rating_q16: u16`

`WeightEntry`（MCS-1 順）:
1. `reviewer_pubkey: bytes[32]`
2. `weight_q32: u32`

`ReputationEntry`（MCS-1 順）:
1. `reviewer_pubkey: bytes[32]`
2. `rep_q32: u32`

`PairMetricEntry`（MCS-1 順）:
1. `reviewer_a: bytes[32]`
2. `reviewer_b: bytes[32]`
3. `metric_q32: i32`

`ClusterMetricEntry`（MCS-1 順）:
1. `cluster_id: bytes[32]`
2. `d_kl_q32: u32`

`SuspectLabelEntry`（MCS-1 順）:
1. `subject_id: bytes[32]`
2. `label: u8`（`1=COLLUSION`, `2=SYBIL`）

構造体（MCS-1 順）:
1. `proof_format_version: u16`（固定値 `1`）
2. `chain_id: u32`
3. `checkpoint_seq: u64`
4. `proposal_id: bytes[32]`
5. `input_commitment: bytes[32]`
6. `metrics_commitment: bytes[32]`
7. `proof_system_id: u16`
8. `proof_bytes: bytes`
9. `worker_pubkey: bytes[32]`
10. `signature: bytes[64]`（Ed25519）
11. `nonce: u64`
12. `expiry_ms: u64`

検証規則:
- `proof_format_version == 1` を **MUST** 満たす。
- `input_commitment` は **MUST** `SHA3-256(MCS-1(EvaluationInputCanonical))` と一致する。
- `metrics_commitment` は **MUST** `SHA3-256(MCS-1(EvaluationMetricsCanonical))` と一致する。
- `signature` は **MUST** `SHA3-256("MISAKA_EVAL_COMMIT_V1" || MCS-1(fields 1..9) || nonce || expiry_ms)` に対して検証成功する。
- `expiry_ms >= now_ms` を **MUST** 満たす（`now_ms` は `02-consensus.md` §6.6 に従う）。
- 同一 `(checkpoint_seq, proposal_id, nonce)` の再利用は **MUST NOT** 許可。
- `proof_system_id` に対応する verifier（`07-crosschain-trust-model.md` の verifier registry）で `proof_bytes` が検証成功しない場合は **MUST** reject。

拒否時エラーコード:
- `ERR_EVAL_COMMITMENT_INVALID`
- `ERR_EVAL_COMMITMENT_EXPIRED`
- `ERR_EVAL_COMMITMENT_NONCE_REUSED`
- `ERR_EVAL_PROOF_VERIFY_FAILED`

## 8. 監査可能性
- 入力データ（ratings, sampled_reviewers, weights, rep）は **MUST** 再現可能に保存。
- `evaluation_commitment_v1` と検証結果（success/reject code）は **MUST** 監査ログに保存する。
- 監査ログは **MUST** `checkpoint_seq` と紐づける。

## 9. 受け入れ基準
1. 4.1〜4.4 のうち 1つでも無効なら **MUST** proposal 評価を停止。
2. 同一入力データで **MUST** 同一 `ProposalScore` を出力。
3. 指標固定化超過時、**MUST** 補正モード遷移が起きる。
4. 同一 `EvaluationInputCanonical` に対して **MUST** 同一 `input_commitment` と `metrics_commitment` を生成する。
5. 無効 `proof_bytes`、期限切れ、nonce 再利用は **MUST** reject される。

## 10. 他仕様参照
- `02-consensus.md`
- `05-storage-layout.md`
- `07-crosschain-trust-model.md`
- `10-tokenomics.md`
- `11-governance-and-emergency-mode.md`
- `13-test-vectors.md`
