# 07 Cross-chain Trust Model 仕様

## 1. 目的と適用範囲
本仕様は Misaka と Solana 間の trust model を定義する。
Phase 1 では semi-trusted モデルを採用し、段階的に trustless へ移行する。

## 2. 規範キーワード
- **MUST**: 準拠実装に必須
- **SHOULD**: 特段の理由がない限り必須
- **MAY**: 任意

## 3. Phase 定義
- **Phase 1 (semi-trusted)**: Misaka finality + External Multisig（M-of-N）で外部実行。
- **Phase 2 (trust-minimized)**: Light client もしくは集約署名検証を導入。
- **Phase 3 (target trustless)**: 外部チェーン上で Misaka finality を暗号学的に単独検証可能。

## 4. Phase 1 信頼境界
### 4.1 信頼前提
Phase 1 では以下を信頼境界とする。
1. Misaka の BFT finality（`02-consensus.md`）
2. Slash Committee の閾値署名（`06-anchor-collateral-phase1.md`）

### 4.2 非目標
- Phase 1 は **MUST NOT** 完全 trustless slashing を主張しない。
- committee 署名なしの自動 slash は **MUST NOT** 実装する。

### 4.3 経済安全定量化（必須）
定義:
- `S_slashable`: Solana 側で slash 実行可能な合計担保量
- `c_sig`: committee 署名者 1 名あたりの最小買収コスト（USD 換算）
- `C_committee = committee_m * c_sig`
- `C_split = (committee_n - committee_m + 1) * c_sig`
- `T_outage`: Solana 停止継続時間

`f_outage(T_outage)` は **MUST** 次を使用。
- `T_outage < 7d` → `1.0`
- `7d <= T_outage < 30d` → `0.7`
- `30d <= T_outage < 90d` → `0.4`
- `T_outage >= 90d` → `0.2`

- `S_effective = S_slashable * f_outage(T_outage)`
- 安全条件として `min(C_committee, C_split) >= k_security * S_effective` を **MUST** 満たす（初期値 `k_security = 1.0`）。

上記条件を満たさない期間、Phase 1 security level は **MUST** `degraded` として扱う。

## 5. 判定主体（曖昧さ排除）
- `slash_proof` の一次整合性（checkpoint/state_root/validator_set_hash）は **MUST** Anchor 側検証ロジックが判定。
- slash 実行承認は **MUST** Slash Committee（M-of-N）が判定。
- Solana Program 実行可否は **MUST** Program 側の on-chain 検証結果で最終決定。

## 6. 署名ルール
### 6.1 最低署名数
- 最低署名数は **MUST** `committee_m` とし、`committee_m >= ceil(2 * committee_n / 3)` を満たす。

### 6.2 署名対象
- 署名対象は **MUST** `06-anchor-collateral-phase1.md` §7.2 で定義された `signing_message` と一致。
- 別メッセージ署名は **MUST** 無効。

### 6.3 失敗時
- 署名不足/無効署名は **MUST** slash 不成立。
- proof 検証失敗は **MUST** 実行拒否。

## 7. Solana 側 proof 受け渡し
### 7.0 version 別 payload
Solana へ渡す payload は `proof_format_version` に応じて **MUST** 次を使用する。
- `proof_format_version = 1`: `slash_proof_v1`（`06-anchor-collateral-phase1.md`）
- `proof_format_version = 2`: `trustless_proof_v2`
- `proof_format_version = 3`: `trustless_proof_v3`

`proof_format_version` の新規有効化（2/3）は **MUST** `11-governance-and-emergency-mode.md` のガバナンス承認 + timelock + final checkpoint 境界適用を経る。

### 7.0.1 verifier registry（必須）
- `proof_system_id` の有効値集合は **MUST** `sys/params/proof_verifier_registry` に保存する。
- `proof_verifier_registry` の各要素は **MUST** 次を含む。
  1. `proof_system_id: u16`
  2. `verifier_id: bytes[32]`
  3. `verifier_code_hash: bytes[32]`
  4. `activated_checkpoint_seq: u64`
  5. `deprecated_after_checkpoint_seq: u64?`
- 未登録 `proof_system_id` は **MUST** reject（`ERR_TRUSTLESS_PROOF_SYSTEM_UNSUPPORTED`）。
- `proof_verifier_registry` 変更は **MUST** `11-governance-and-emergency-mode.md` §5 の timelock フローを経る。
- Program は **MUST** v2/v3 検証時、`proof_system_id` に対応し、`activated_checkpoint_seq <= checkpoint_seq` かつ（`deprecated_after_checkpoint_seq` 未設定または `checkpoint_seq < deprecated_after_checkpoint_seq`）を満たす要素を選び、実行時 verifier の code hash が `verifier_code_hash` と一致することを検証する。

### 7.1 `trustless_proof_v2`（Phase 2）
構造体（MCS-1 順）:
1. `proof_format_version: u16`（固定値 `2`）
2. `chain_id: u32`
3. `checkpoint_seq: u64`
4. `checkpoint_digest: bytes[32]`
5. `state_root: bytes[32]`
6. `validator_set_hash: bytes[32]`
7. `finality_proof_v1_bytes: bytes`
8. `proof_system_id: u16`
9. `proof_bytes: bytes`
10. `nonce: u64`
11. `expiry_ms: u64`

検証規則:
- `proof_format_version == 2` を **MUST** 満たす。
- `finality_proof_v1_bytes` は **MUST** `02-consensus.md` §4.3 の全検証規則に一致する。
- デコード後 `finality_proof_v1.checkpoint_digest == checkpoint_digest` を **MUST** 満たす。
- デコード後 `finality_proof_v1.validator_set_hash == validator_set_hash` を **MUST** 満たす。
- `checkpoint_digest`、`state_root`、`validator_set_hash` は **MUST** 当該 `checkpoint_seq` の確定値と一致する。
- `proof_bytes` は **MUST** `proof_system_id` 対応 verifier で検証成功する。
- `expiry_ms >= now_ms` を **MUST** 満たす（`now_ms` は `02-consensus.md` §6.6 に従う）。
- 同一 `(checkpoint_seq, nonce)` の再利用は **MUST NOT** 許可。

### 7.2 `trustless_proof_v3`（Phase 3）
構造体（MCS-1 順）:
1. `proof_format_version: u16`（固定値 `3`）
2. `chain_id: u32`
3. `checkpoint_seq: u64`
4. `checkpoint_digest: bytes[32]`
5. `state_root: bytes[32]`
6. `validator_set_hash: bytes[32]`
7. `proof_system_id: u16`
8. `public_inputs_commitment: bytes[32]`
9. `proof_bytes: bytes`
10. `nonce: u64`
11. `expiry_ms: u64`

検証規則:
- `proof_format_version == 3` を **MUST** 満たす。
- `public_inputs_commitment` は **MUST** `SHA3-256(MCS-1(chain_id, checkpoint_seq, checkpoint_digest, state_root, validator_set_hash))` と一致する。
- `proof_bytes` は **MUST** `proof_system_id` 対応 verifier で検証成功する。
- `checkpoint_digest`、`state_root`、`validator_set_hash` は **MUST** 当該 `checkpoint_seq` の確定値と一致する。
- `expiry_ms >= now_ms` を **MUST** 満たす（`now_ms` は `02-consensus.md` §6.6 に従う）。
- 同一 `(checkpoint_seq, nonce)` の再利用は **MUST NOT** 許可。

### 7.3 Program 検証手順（共通）
Program は **MUST** version 別に以下を検証する。
1. `proof_format_version` が `1/2/3` のいずれか
2. `chain_id` が実行対象 chain と一致
3. `checkpoint_seq` 対応の `checkpoint_digest/state_root/validator_set_hash` 一致
4. nonce 再利用なし（v1 は `(checkpoint_seq, accused_validator_pubkey, nonce)`、v2/v3 は `(checkpoint_seq, nonce)`）
5. `expiry_ms` 未超過
6. version 固有検証（v1: committee 閾値署名, v2/v3: `proof_system_id` 対応 verifier 成功）

拒否時エラーコード:
- `ERR_TRUSTLESS_PROOF_VERSION_UNSUPPORTED`
- `ERR_TRUSTLESS_PROOF_SYSTEM_UNSUPPORTED`
- `ERR_TRUSTLESS_PROOF_VERIFY_FAILED`
- `ERR_TRUSTLESS_VERIFIER_HASH_MISMATCH`
- `ERR_TRUSTLESS_PROOF_NONCE_REUSED`
- `ERR_TRUSTLESS_PROOF_EXPIRED`

## 8. Phase 2 への移行条件
Phase 1 から Phase 2 への移行は、以下全条件を **MUST** 満たす。
1. Light client 仕様または集約署名仕様が固定済み
2. 少なくとも 2 実装で相互検証テストに合格
3. 悪性 proof テスト（偽state_root/偽validator_set_hash）で拒否を確認
4. ガバナンス承認 + timelock 経過 + final checkpoint 境界適用

### 8.1 Adversarial Simulation 受け入れ基準（必須）
- committee 買収シナリオ、committee 分裂シナリオ、Solana 停止シナリオ（7d/30d/90d）を **MUST** シミュレーションする。
- 各シナリオで `min(C_committee, C_split) / S_effective` を **MUST** 出力し、`>= 1.0` を維持できない場合は Phase 2 移行を **MUST NOT** 承認する。
- `trustless_proof_v2` / `trustless_proof_v3` それぞれについて、偽 `checkpoint_digest` / 偽 `state_root` / 偽 `validator_set_hash` / nonce 再利用 / 期限切れを入力した reject 試験を **MUST** 実施する。
- `trustless_proof_v2` / `trustless_proof_v3` のテスト入力/期待結果は **MUST** `13-test-vectors.md` 形式で保存する。
- シミュレーション入力/出力は **MUST** 監査ログに保存し、`checkpoint_seq` と紐付ける。

## 9. trustless claim 許可条件
「trustless」を名乗るのは以下成立時のみ **MUST** 許可。
- 外部チェーン上で Misaka finality を committee 非依存で検証可能
- 検証に必要な証跡が公開仕様で再現可能
- 監査済み実装が mainnet に反映済み

上記未達時は **MUST** `semi-trusted` 表記を維持する。

## 10. 他仕様参照
- `02-consensus.md`
- `03-deterministic-execution.md`
- `06-anchor-collateral-phase1.md`
- `10-tokenomics.md`
- `11-governance-and-emergency-mode.md`
- `13-test-vectors.md`
- `16-bft-liveness-fallback.md`
