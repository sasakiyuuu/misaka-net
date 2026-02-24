# 13 Test Vectors 仕様（決定性検証）

## 1. 目的と適用範囲
本仕様は Misaka の決定性を実装間で検証するためのテストベクタ形式と必須ケースを定義する。
対象は以下。
- `checkpoint_tx_digest_list` の順序決定性
- `tx_digest_merkle_root` 決定性
- `state_root`（StateRoot-v1 / StateRoot-v2）
- tombstone / `sys/params/*` / empty state ケース
- `07-crosschain-trust-model.md` の `trustless_proof_v2` / `trustless_proof_v3` reject ケース
- `12-proposal-evaluation-security.md` の `evaluation_commitment_v1` 検証ケース

## 2. 規範キーワード
- **MUST**: 準拠実装に必須
- **SHOULD**: 特段の理由がない限り必須
- **MAY**: 任意

## 3. 共通前提
- シリアライズは **MUST** `MCS-1` を使用（`01-tx-object-checkpoint.md`）。
- ハッシュは **MUST** `SHA3-256`。
- 順序検証は **MUST** `03-deterministic-execution.md` の `DET_ORDER_V1` を使用。
- StateRoot-v1 は **MUST** `05-storage-layout.md` に一致。
- StateRoot-v2 は **MUST** `08-state-commitment-and-snapshots.md` の JMT 規則に一致。

## 4. ベクタファイル形式（JSON）
```json
{
  "vector_id": "string",
  "state_root_version": 1,
  "inputs": {
    "objects": [{"object_bytes_hex": "..."}],
    "version_index_latest_only": [{"key_hex": "...", "value_hex": "..."}],
    "sys_params": [{"key_hex": "...", "value_hex": "..."}],
    "txs": [{"tx_bytes_hex": "..."}]
  },
  "checkpoint": {
    "checkpoint_header_bytes_hex": "...",
    "checkpoint_tx_digest_list": ["hex32", "hex32"]
  },
  "expected": {
    "tx_digest_merkle_root_hex": "hex32",
    "state_root_hex": "hex32",
    "tx_outcomes": ["success", "fail"]
  }
}
```

形式規則:
- `object_bytes_hex` は **MUST** `MCS-1(Object)`。
- `tx_bytes_hex` は **MUST** `MCS-1(Transaction)`。
- `checkpoint_header_bytes_hex` は **MUST** `MCS-1(CheckpointHeader)`。
- `checkpoint_tx_digest_list` は **MUST** `DET_ORDER_V1` 順序。

## 5. 必須ベクタ

### 5.0 Vector O2（順序責務不一致 / DET_ORDER_V1）
- consensus が生成した `checkpoint_tx_digest_list` を意図的に 1 要素入れ替えた入力を与える
- 期待: execution 側の `verify_det_order_v1` で reject
- 期待エラー: `ERR_DET_ORDER_MISMATCH`

### 5.1 Vector E0（empty / v1）
- `state_root_version=1`
- `objects=[]`, `version_index_latest_only=[]`, `sys_params=[]`, `txs=[]`
- 期待:
  - `state_root = SHA3-256(empty bytes)`
  - `tx_digest_merkle_root = SHA3-256(empty bytes)`

### 5.2 Vector T1（tombstone / v1）
- tombstone object を 1 件含む（`lifecycle_state=1`, `payload=empty`）
- `version_index_latest_only` に tombstone 対応 entry を含む
- 期待: 実装間で `state_root` 一致

### 5.3 Vector P1（sys/params変更 / v1）
- `sys/params/state_root_version`
- `sys/params/weak_subjectivity_period_ms`
- `sys/params/r_annual`
を含む
- 期待: 実装間で `state_root` 一致

### 5.4 Vector V2-0（empty / v2）
- `state_root_version=2`
- `kv_entries=[]`
- 期待: `state_root = ZERO_HASH`（32-byte zero）

### 5.5 Vector V2-1（tombstone + sys/params / v2）
- `state_root_version=2`
- tombstone と `sys/params/*` を同時に含む
- 期待: JMT ルールで `state_root` 一致
- 期待: path 衝突入力は reject

### 5.6 Vector O1（競合順序 / DET_ORDER_V1）
- shared tx / owned-only tx / mixed tx を混在
- `checkpoint_tx_digest_list` が `DET_ORDER_V1` に一致
- 期待:
  - `tx_outcomes` 実装間一致
  - `state_root` 実装間一致

## 6. 実行・検証規則
- 実装は **MUST** 各ベクタについて以下を検証。
  1. `tx_hash` 再計算
  2. `checkpoint_tx_digest_list` が `DET_ORDER_V1` 一致
  3. `tx_digest_merkle_root` 再計算
  4. `state_root` 再計算
  5. `expected` との一致

## 7. Cross-chain / Evaluation Commitment ベクタ形式（JSON）
### 7.1 trustless proof vector
```json
{
  "vector_id": "string",
  "category": "trustless_proof_v2|trustless_proof_v3",
  "checkpoint_seq": 0,
  "inputs": {
    "proof_format_version": 0,
    "proof_system_id": 0,
    "finality_proof_v1_bytes_hex": "...",
    "public_inputs_commitment_hex": "hex32",
    "proof_bytes_hex": "...",
    "public_inputs": {
      "chain_id": 0,
      "checkpoint_digest_hex": "hex32",
      "state_root_hex": "hex32",
      "validator_set_hash_hex": "hex32"
    },
    "nonce": 0,
    "expiry_ms": 0
  },
  "expected": {
    "result": "accept|reject",
    "error_code": "ERR_*"
  }
}
```

### 7.2 evaluation commitment vector
```json
{
  "vector_id": "string",
  "category": "evaluation_commitment_v1",
  "checkpoint_seq": 0,
  "inputs": {
    "proof_format_version": 1,
    "chain_id": 0,
    "checkpoint_seq": 0,
    "proposal_id_hex": "hex32",
    "input_commitment_hex": "hex32",
    "metrics_commitment_hex": "hex32",
    "proof_system_id": 0,
    "proof_bytes_hex": "...",
    "worker_pubkey_hex": "hex32",
    "signature_hex": "hex64",
    "nonce": 0,
    "expiry_ms": 0
  },
  "expected": {
    "result": "accept|reject",
    "error_code": "ERR_*"
  }
}
```

形式規則:
- `proof_bytes_hex` は **MUST** `MCS-1` bytes（proof payload 本体）を使用。
- `expected.error_code` は reject 時に **MUST** 指定。

## 8. 失敗条件
- `checkpoint_tx_digest_list` 不一致: **MUST** reject（`ERR_DET_ORDER_MISMATCH`）
- `tx_digest_merkle_root` 不一致: **MUST** reject（`ERR_TX_MERKLE_MISMATCH`）
- `state_root` 不一致: **MUST** fail（`ERR_STATE_ROOT_MISMATCH`）
- state_root v2 path 重複: **MUST** reject（`ERR_STATE_ROOT_DUPLICATE_PATH`）
- gas payment object 不正: **MUST** reject（`ERR_GAS_PAYMENT_OBJECT_INVALID`）
- gas 会計 overflow: **MUST** reject（`ERR_GAS_ACCOUNTING_OVERFLOW`）
- fork finality conflict: **MUST** reject（`ERR_FORK_FINALITY_CONFLICT`）
- fork validator set mismatch: **MUST** reject（`ERR_FORK_VALIDATOR_SET_MISMATCH`）
- fork commit power 不足: **MUST** reject（`ERR_FORK_INSUFFICIENT_COMMIT_POWER`）
- trustless proof version 未対応: **MUST** reject（`ERR_TRUSTLESS_PROOF_VERSION_UNSUPPORTED`）
- trustless proof system 未登録: **MUST** reject（`ERR_TRUSTLESS_PROOF_SYSTEM_UNSUPPORTED`）
- trustless proof 検証失敗: **MUST** reject（`ERR_TRUSTLESS_PROOF_VERIFY_FAILED`）
- trustless verifier hash 不一致: **MUST** reject（`ERR_TRUSTLESS_VERIFIER_HASH_MISMATCH`）
- trustless proof nonce 再利用: **MUST** reject（`ERR_TRUSTLESS_PROOF_NONCE_REUSED`）
- trustless proof 期限切れ: **MUST** reject（`ERR_TRUSTLESS_PROOF_EXPIRED`）
- evaluation commitment 無効: **MUST** reject（`ERR_EVAL_COMMITMENT_INVALID`）
- evaluation commitment proof 検証失敗: **MUST** reject（`ERR_EVAL_PROOF_VERIFY_FAILED`）
- evaluation commitment 期限切れ: **MUST** reject（`ERR_EVAL_COMMITMENT_EXPIRED`）
- evaluation commitment nonce 再利用: **MUST** reject（`ERR_EVAL_COMMITMENT_NONCE_REUSED`）

## 9. 他仕様参照
- `01-tx-object-checkpoint.md`
- `03-deterministic-execution.md`
- `05-storage-layout.md`
- `07-crosschain-trust-model.md`
- `08-state-commitment-and-snapshots.md`
- `12-proposal-evaluation-security.md`
