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
- reviewer は **MUST** epoch ごとに再抽選。
- 抽選シードは **MUST** `validator_set_hash || checkpoint_seq` を含む。
- 同一入力で抽選結果は **MUST** 再現可能。

### 4.4 reputation decay（必須）
- reputation は **MUST** 時間減衰。
- 式: `rep_t = rep_0 * exp(-lambda * t)`
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

## 6. 指標固定化回避（必須）
- 単一利害関係者カテゴリ（組織・地域・資本属性）が合計重み `> 35%` となる状態を **MUST NOT** 継続許容。
- 連続 `3` epoch で超過した場合、システムは **MUST** 補正モードへ移行。

補正モード要件:
1. 超過カテゴリの重みを減衰
2. 非超過カテゴリのサンプリング比率を増加
3. 補正履歴を監査ログへ記録

## 7. 不正検知
- 相関が高い投票クラスターは **MUST** collusion 検査対象。
- Sybil 疑い ID は **MUST** 一時隔離し、評価重みを 0 に設定。
- 検知/解除ルールは **MUST** `sys/params/*` に保存し、ガバナンス変更のみ許可。

## 8. 監査可能性
- 入力データ（ratings, sampled_reviewers, weights, rep）は **MUST** 再現可能に保存。
- 監査ログは **MUST** `checkpoint_seq` と紐づける。

## 9. 受け入れ基準
1. 4.1〜4.4 のうち 1つでも無効なら **MUST** proposal 評価を停止。
2. 同一入力データで **MUST** 同一 `ProposalScore` を出力。
3. 指標固定化超過時、**MUST** 補正モード遷移が起きる。

## 10. 他仕様参照
- `10-tokenomics.md`
- `11-governance-and-emergency-mode.md`
- `02-consensus.md`
- `05-storage-layout.md`
