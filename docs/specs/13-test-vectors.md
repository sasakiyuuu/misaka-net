# 13 Test Vectors 仕様（決定性検証）

## 1. 目的と適用範囲
本仕様は Misaka の決定性を実装間で検証するためのテストベクタ形式と必須ケースを定義する。
対象は以下。
- `checkpoint_tx_digest_list` の順序決定性
- `tx_digest_merkle_root` 決定性
- `state_root`（StateRoot-v1 / StateRoot-v2）
- tombstone / `sys/params/*` / empty state ケース

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

### 5.4 Vector V2-1（tombstone + sys/params / v2）
- `state_root_version=2`
- tombstone と `sys/params/*` を同時に含む
- 期待: JMT ルールで `state_root` 一致

### 5.5 Vector O1（競合順序 / DET_ORDER_V1）
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

## 7. 失敗条件
- `checkpoint_tx_digest_list` 不一致: **MUST** reject（`ERR_DET_ORDER_MISMATCH`）
- `tx_digest_merkle_root` 不一致: **MUST** reject（`ERR_TX_MERKLE_MISMATCH`）
- `state_root` 不一致: **MUST** fail（`ERR_STATE_ROOT_MISMATCH`）

## 8. 他仕様参照
- `01-tx-object-checkpoint.md`
- `03-deterministic-execution.md`
- `05-storage-layout.md`
- `08-state-commitment-and-snapshots.md`
