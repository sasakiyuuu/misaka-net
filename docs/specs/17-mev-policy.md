# 17 MEV Policy 仕様

## 1. 目的と適用範囲
本仕様は Misaka における MEV の許容範囲、非許容行為、透明性要件を定義する。
本仕様は順序決定性（`DET_ORDER_V1`）を不変条件として扱う。

## 2. 規範キーワード
- **MUST**: 準拠実装に必須
- **SHOULD**: 特段の理由がない限り必須
- **MAY**: 任意

## 3. 不変条件
- `checkpoint_tx_digest_list` は **MUST** `DET_ORDER_V1` に一致。
- 実行層は **MUST NOT** `checkpoint_tx_digest_list` を再順序化。
- proposer/builder は **MUST NOT** 上記不変条件を破る。

## 4. 許容行為
- mempool 受入優先度の調整（`04-resource-limits.md` の範囲内）は **MAY**。
- fee 最適化提案は **MAY**。

## 5. 非許容行為
- `DET_ORDER_V1` 逸脱を狙う任意の reorder は **MUST NOT**。
- private bundle を使った順序上書きは **MUST NOT**。
- 検証不能な off-chain side agreement による順序固定は **MUST NOT**。

### 5.1 経済的強制（必須）
- proposer は **MUST** `04-resource-limits.md` の `effective_fee_per_byte` 優先規則と整合する候補集合から block を作る。
- `PRIORITY_WINDOW = 1_000` 件の高優先度 TX 集合を **MUST** 評価対象とする。
- 検証可能な無効理由（期限切れ/検証失敗）を除き、`PRIORITY_WINDOW` からの除外率は **MUST** `<= 1%`。
- 除外率 `> 1%` は **MUST** `ERR_MEV_POLICY_VIOLATION`。

## 6. 透明性要件
- 各 block/ checkpoint について以下を **MUST** 記録。
  - `ordered_tx_list` のハッシュ
  - `checkpoint_tx_digest_list`
  - reject された高優先度 TX の件数
  - `priority_window_hash = SHA3-256(concat(tx_hashes in PRIORITY_WINDOW))`
  - `priority_exclusion_rate`
- 監査可能ログは **SHOULD** 公開 API で取得可能にする。

## 7. 監査・違反対応
- MEV違反検知時は **MUST** `ERR_MEV_POLICY_VIOLATION` を生成。
- 重大違反は **MUST** ガバナンス審査対象とし、必要に応じ slashing 判断へ引き渡す。

## 8. ガバナンス変更
- MEV policy パラメータ変更は **MUST** ガバナンス + timelock 経由。
- 緊急モード中は **MUST NOT** 恒久パラメータを変更。

## 9. 他仕様参照
- `02-consensus.md`
- `03-deterministic-execution.md`
- `04-resource-limits.md`
- `10-tokenomics.md`
- `11-governance-and-emergency-mode.md`
- `15-block-limits.md`
