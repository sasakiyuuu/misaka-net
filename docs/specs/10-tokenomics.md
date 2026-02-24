# 10 Tokenomics 仕様（Phase 4）

## 1. 目的と適用範囲
本仕様は Misaka の発行量、インフレ、報酬、slashing、proposal grant 原資を定義する。

## 2. 規範キーワード
- **MUST**: 準拠実装に必須
- **SHOULD**: 特段の理由がない限り必須
- **MAY**: 任意

## 3. 参照用語
- `checkpoint_seq`, `state_root`, `validator_set_hash`, `MCS-1` は 01-05 に従う。

## 4. インフレモデル
### 4.1 変数
- `S_t`: epoch t 開始時の循環供給
- `r_annual`: 年率インフレ
- `EPOCHS_PER_YEAR = 365`
- `r_epoch = r_annual / EPOCHS_PER_YEAR`
- `safe_mode_active`: `16-bft-liveness-fallback.md` で定義される safe mode 状態
- `long_stall_active`: safe mode が `T_STALL_LONG = 7 days` 以上継続した状態
- `r_epoch_safe = 0`（safe mode 中の既定値）

### 4.2 式
通常時:
`Inflation_t = S_t * r_epoch`

safe mode 時:
`Inflation_t = S_t * r_epoch_safe`

初期値:
- `r_annual = 0.07`
- `r_epoch_safe = 0`

## 5. 報酬配分
### 5.1 全体配分
- `p_validator = 0.80`
- `p_grants = 0.15`
- `p_treasury = 0.05`

制約:
- `p_validator + p_grants + p_treasury = 1.0` を **MUST** 満たす。

### 5.2 validator 報酬式
通常時:
`Reward_i = Inflation_t * p_validator * (W_i / W_total) * availability_i`

safe mode / long-stall 時:
`Reward_i = 0`

- `W_i`: validator i の voting power
- `W_total`: active validator 全体の voting power 合計（`sum(W_i)`）
- `availability_i`: epoch 内 precommit 参加率（0..1）

## 6. slashing penalty

> フェーズ境界: Phase 1〜3 のクロスチェーン slashing 実行は `06-anchor-collateral-phase1.md` / `07-crosschain-trust-model.md` に従う。Phase 4 は本仕様の経済計算を規定する。

### 6.0 Cross-chain 経済リンク（必須）
- `S_slashable = sum(Stake_i)`（外部チェーンで slash 実行可能な担保合計）
- `S_slashable` は **MUST** epoch ごとに監査ログへ記録し、`07-crosschain-trust-model.md` の安全条件評価に使用する。

### 6.1 種別
- `DOUBLE_SIGN`
- `DOWNTIME`
- `INVALID_PROPOSAL`

### 6.2 式
`Slash_i = Stake_i * penalty_rate(type)`

初期値:
- `penalty_rate(DOUBLE_SIGN)=0.05`
- `penalty_rate(DOWNTIME)=0.01`
- `penalty_rate(INVALID_PROPOSAL)=0.02`

### 6.3 適用
- slashing 適用は **MUST** final checkpoint 境界で行う。
- fallback mode（`11-governance-and-emergency-mode.md`）中は slashing を **MUST** 保留し、解除後最初の final checkpoint 境界で適用する。
- 結果は **MUST** `state_root` に反映される。

## 7. Proposal grant 原資と上限
### 7.1 原資
通常時:
`GrantPool_t = Inflation_t * p_grants`

safe mode / long-stall 時:
`GrantPool_t = 0`

### 7.2 上限
- 1 proposal 上限: `0.50% * Inflation_t`
- 1 epoch 総額上限: `GrantPool_t`

## 8. 8GB Validator Profitability Check（必須）
定義:
- `C_8GB_epoch`: 8GB validator の epoch あたり運用コスト（USD 換算）
- `Fees_i`: validator i の epoch 手数料収入
- `Profit_i = Reward_i + Fees_i - C_8GB_epoch`

受け入れ基準:
- 標準負荷（`04` + `15` 上限）で、アクティブ validator の中央値 `Profit_i` は **MUST** `>= 0`。
- 中央値 `Profit_i < 0` が `3` epoch 連続した場合、`profitability_warning` を **MUST** 発行し、パラメータ見直し proposal を **MUST** 要求する。

## 9. パラメータ更新
- 全パラメータは **MUST** `sys/params/*` に MCS-1 で保存。
- 更新は **MUST** `11-governance-and-emergency-mode.md` の timelock を経る。

## 10. 他仕様参照
- `02-consensus.md`
- `04-resource-limits.md`
- `05-storage-layout.md`
- `11-governance-and-emergency-mode.md`
- `12-proposal-evaluation-security.md`
- `15-block-limits.md`
- `16-bft-liveness-fallback.md`
