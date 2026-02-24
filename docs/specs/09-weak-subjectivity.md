# 09 Weak Subjectivity 仕様

## 1. 目的と適用範囲
本仕様は long-range 攻撃耐性のための weak subjectivity（WS）運用を定義する。
対象は新規ノード起動、trusted checkpoint 要件、stale checkpoint 拒否条件。

## 2. 規範キーワード
- **MUST**: 準拠実装に必須
- **SHOULD**: 特段の理由がない限り必須
- **MAY**: 任意

## 3. 定義
- `trusted_checkpoint`: 起動時に信頼アンカーとして使用する checkpoint
- `WSP`（Weak Subjectivity Period）: trusted checkpoint の許容鮮度期間
- `stale_checkpoint`: `now_ms - checkpoint.timestamp_ms > WSP` を満たす checkpoint

## 4. WSP パラメータ
- 初期値: `WSP_MS = 14 days`
- `sys/params/weak_subjectivity_period_ms` が存在する場合、**MUST** それを優先。

## 5. trusted checkpoint 要件
trusted checkpoint は **MUST** 以下を含む。
1. `CheckpointHeader`（MCS-1 bytes）
2. `checkpoint_digest`
3. `state_root`
4. `validator_set_hash`
5. `finality_proof_v1_bytes`（`02-consensus.md` §4.3 の MCS-1 bytes）

検証:
- `checkpoint_digest` 再計算一致
- `finality_proof_v1_bytes` をデコードし、`02-consensus.md` §4.3 の全検証規則に一致
- `finality_proof_v1.validator_set_hash` が trusted checkpoint の `validator_set_hash` と一致
- `state_root` がローカル再計算と一致

## 6. stale checkpoint 起動拒否
定義:
- `hardcoded_min_checkpoint_seq` はクライアントに同梱される最小許容 trusted checkpoint（hard-coded）値。

以下のいずれかでノードは **MUST** 起動拒否。
1. `trusted_checkpoint` が stale
2. `trusted_checkpoint.checkpoint_seq < hardcoded_min_checkpoint_seq`
3. `validator_set_hash` 不整合

拒否時エラーコード:
- `ERR_WS_STALE_CHECKPOINT`
- `ERR_WS_TOO_OLD_CHECKPOINT`
- `ERR_WS_VALIDATOR_SET_MISMATCH`

## 7. hard-coded checkpoint 更新ポリシー
- クライアントは **MUST** hard-coded trusted checkpoint を保持。
- 更新時は **MUST** 以下を満たす。
  1. finality 証明付き
  2. 既存 hard-coded checkpoint より新しい
  3. 署名検証・root 検証済み
- 更新は **SHOULD** 定期リリースで配布。

## 8. 起動手順（snapshot + replay）
1. trusted checkpoint 読み込み
2. WS 条件検証
3. snapshot 取得と検証（`08` 参照）
4. `checkpoint_seq + 1` から replay
5. 最新 final checkpoint まで追随

## 9. 運用要件
- ノード運用者は **SHOULD** WSP の 1/2 以内で trusted checkpoint を更新する。
- 監査ログは **MUST** 起動時検証結果を保存する。

## 10. 他仕様参照
- `01-tx-object-checkpoint.md`
- `02-consensus.md`
- `08-state-commitment-and-snapshots.md`
- `11-governance-and-emergency-mode.md`
