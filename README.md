# Misaka Net Specs

Misaka の仕様策定リポジトリです。
このリポジトリは「実装可能な仕様中心（仕様90% / 思想10%）」で、`docs/specs` に段階的な規範仕様をまとめています。

## 見方（推奨読了順）

### Phase 0–1: Core Spec Freeze（最優先）
1. `docs/specs/01-tx-object-checkpoint.md`
   TX / Object / Checkpoint の canonical 定義
2. `docs/specs/02-consensus.md`
   CometBFT/Tendermint 系の finality・epoch・validator set 更新
3. `docs/specs/03-deterministic-execution.md`
   shared object を含む決定論的実行順序
4. `docs/specs/04-resource-limits.md`
   8GB validator 前提の DoS 制限・安全上限
5. `docs/specs/05-storage-layout.md`
   validator/indexer 分離と state root 対象の固定

### Phase 2: Anchor 担保 PoS（現実路線）
6. `docs/specs/06-anchor-collateral-phase1.md`
   Anchor は担保管理中心、slash は finality + M-of-N 承認
7. `docs/specs/07-crosschain-trust-model.md`
   Phase 1 semi-trusted から trustless への移行条件

### Phase 3: State Commitment / Weak Subjectivity
8. `docs/specs/08-state-commitment-and-snapshots.md`
   StateRoot-v1/v2（JMT）移行、snapshot 構成と検証
9. `docs/specs/09-weak-subjectivity.md`
   WS period、trusted checkpoint、stale checkpoint 拒否

### Phase 4: Economics / Governance Guardrails
10. `docs/specs/10-tokenomics.md`
    インフレ・報酬・slashing・grant 原資
11. `docs/specs/11-governance-and-emergency-mode.md`
    権限分離、timelock、fallback mode、local override
12. `docs/specs/12-proposal-evaluation-security.md`
    collusion/Sybil/bootstrap bias 対策

## 仕様の読み方ルール
- 規範語: **MUST / SHOULD / MAY**
- 合意クリティカルな値（`state_root`, `checkpoint_seq`, `validator_set_hash`）は各仕様の定義を優先
- `sys/params/*` の更新はガバナンスと timelock 経由

## 実装へ落とす時の最短動線
1. 01→03→05 を先に実装し、同一入力同一 `state_root` をテスト
2. 04 の上限値を enforce して 8GB envelope を維持
3. 06/07 を使って Phase 1 の Anchor 連携を実装
4. 08/09 で snapshot 同期と WS 運用を実装
5. 10〜12 で経済・ガバナンス・評価安全性を適用
