# 01 Tx / Object / Checkpoint 仕様（Phase 0）

## 1. 目的と適用範囲
本仕様は Misaka の Transaction（TX）、Object、Checkpoint の最小規範を固定する。
本仕様の対象は合意クリティカルであり、準拠実装は本仕様に従わなければならない。

## 2. 規範キーワード
- **MUST**: 準拠実装に必須
- **SHOULD**: 特段の理由がない限り必須
- **MAY**: 任意

## 3. 共通定義

### 3.1 バイト順序と比較
- 整数型は **MUST** little-endian でエンコードする。
- バイト列の昇順比較は **MUST** unsigned bytewise lexicographic order（辞書順）を用いる。

### 3.2 MCS-1（Misaka Canonical Serialization v1）
MCS-1 は以下を **MUST** 満たす。
- フィールド順序固定
- 可変長データの長さ前置
- 並び順が未定義な map/set の禁止

MCS-1 型規則:
- `u8/u16/u32/u64`: fixed width, little-endian
- `bool`: `0x00`（false）/`0x01`（true）
- `bytes`: `u32(length)` + raw bytes
- `string`: UTF-8 bytes を `bytes` と同形式でエンコード
- `list<T>`: `u32(count)` + 要素を宣言順に連結

### 3.3 ハッシュ
- ハッシュ関数は **MUST** `SHA3-256` を用いる。
- 構造体ダイジェストは **MUST** `SHA3-256(MCS-1(encoded_struct))` とする。

### 3.4 用語
- `tx_hash` と `tx_digest` は同義であり、`SHA3-256(MCS-1(Transaction))` を指す。
- `tx_digest_list` と `checkpoint_tx_digest_list` は同義であり、Checkpoint に格納される実行順序確定済み TX ダイジェスト列を指す。
- `ObjectRef` は `object_id + version + digest` の三つ組。

## 4. Transaction 定義

### 4.1 Transaction フィールド（シリアライズ順）
1. `tx_format_version: u16`（固定値 `1`）
2. `chain_id: u32`
3. `sender: bytes[32]`
4. `nonce: u64`
5. `gas_budget: u64`
6. `gas_price: u64`
7. `gas_payment_object_id: bytes[32]`
8. `expiration_checkpoint: u64`
9. `inputs: list<InputRef>`
10. `actions: list<Action>`
11. `signature_scheme: u8`（`1=Ed25519`）
12. `signature: bytes[64]`（Ed25519 固定長）

### 4.2 InputRef フィールド（シリアライズ順）
1. `object_id: bytes[32]`
2. `kind: u8`（`0=Owned`, `1=Shared`, `2=Immutable`）
3. `access: u8`（`0=ReadOnly`, `1=Mutable`）
4. `expected_version_present: bool`
5. `expected_version: u64`（present=true の場合のみ）
6. `expected_digest_present: bool`
7. `expected_digest: bytes[32]`（present=true の場合のみ）

制約:
- `kind=Shared` の場合、`expected_version_present=false` かつ `expected_digest_present=false` を **MUST** 満たす。
- `kind=Owned` または `kind=Immutable` の場合、`expected_version_present=true` かつ `expected_digest_present=true` を **MUST** 満たす。

### 4.3 Action フィールド（定義必須）
`Transaction.actions` の要素 `Action` は **MUST** 次の tagged-union 形式を使用する（MCS-1 順）。

共通先頭:
1. `action_kind: u8`

`action_kind=1`（`TRANSFER_OBJECT`）:
2. `object_id: bytes[32]`
3. `to_address: bytes[32]`

`action_kind=2`（`MOVE_CALL`）:
2. `package_id: bytes[32]`
3. `module_name: string`
4. `function_name: string`
5. `type_args: list<string>`
6. `arg_refs: list<u16>`（`inputs` インデックス参照）

`action_kind=3`（`PUBLISH_PACKAGE`）:
2. `package_bytes: bytes`

`action_kind=4`（`DELETE_OBJECT`）:
2. `object_id: bytes[32]`

`action_kind` が未定義値の場合、TX は **MUST** reject（`ERR_ACTION_KIND_UNSUPPORTED`）。

### 4.4 Transaction 妥当性
- 同一 TX バイト列は **MUST** 同一 `tx_hash` を生成する。
- `signature_scheme=1` の場合、署名検証は **MUST** Ed25519 で行う。
- `current_checkpoint` は「実行開始時点の最新 finalized `checkpoint_seq`」を指す。
- `expiration_checkpoint < current_checkpoint` の TX は **MUST** 無効。
- `actions` は **MUST** 1 件以上を含む。
- `MOVE_CALL.arg_refs` の各要素は **MUST** `len(inputs)` 未満。
- `gas_payment_object_id` は **MUST** `kind=Owned` かつ `access=Mutable` の input と一致する。
- `gas_payment_object_id` は **MUST** `sender` 所有 Object でなければならない。
- `gas_payment_object_id` 条件違反 TX は **MUST** reject（`ERR_GAS_PAYMENT_OBJECT_INVALID`）。

## 5. Object 定義

### 5.1 Object フィールド（シリアライズ順）
1. `object_format_version: u16`（固定値 `1`）
2. `object_id: bytes[32]`
3. `version: u64`
4. `owner_kind: u8`（`0=AddressOwner`, `1=ObjectOwner`, `2=SharedOwner`, `3=Immutable`）
5. `owner_data: bytes`
6. `type_tag: string`
7. `payload: bytes`
8. `lifecycle_state: u8`（`0=Live`, `1=Tombstone`）

`digest` は保存フィールドではなく、**MUST** `SHA3-256(MCS-1(Object))` で算出する。

### 5.2 owner_data 形式
- `owner_kind=AddressOwner`: `bytes[32]`（address）
- `owner_kind=ObjectOwner`: `bytes[32]`（object_id）
- `owner_kind=SharedOwner`: `u64(initial_shared_version)`
- `owner_kind=Immutable`: `bytes` 長さ 0

### 5.3 Versioning ルール
- 新規作成 Object の `version` は **MUST** `1`。
- Mutable Object（Owned/Shared）の成功更新は **MUST** `version+1`。
- 同一 TX 内で同一 Object を複数回更新しても version 増加は **MUST** 1 回。
- 削除は **MUST** Tombstone レコードを書き、`version+1`。
- Tombstone レコードは **MUST** `payload=empty bytes` かつ `lifecycle_state=1`。
- Immutable Object は **MUST NOT** 更新される。

### 5.4 参照整合
- Owned/Immutable 参照は **MUST** `expected_version` と `expected_digest` が現在値に一致する。
- 不一致は **MUST** `ERR_OBJECT_VERSION_MISMATCH` で失敗。

## 6. Checkpoint 定義

### 6.1 Checkpoint Header フィールド（シリアライズ順）
1. `checkpoint_format_version: u16`（固定値 `1`）
2. `chain_id: u32`
3. `epoch: u64`
4. `checkpoint_seq: u64`
5. `prev_checkpoint_digest: bytes[32]`
6. `timestamp_ms: u64`
7. `tx_count: u32`
8. `tx_digest_merkle_root: bytes[32]`
9. `state_root: bytes[32]`
10. `validator_set_hash: bytes[32]`
11. `tx_ordering_rule_id: u16`（固定値 `1` = `DET_ORDER_V1`）
12. `execution_rule_id: u16`（固定値 `1` = `DET_EXEC_V1`）

### 6.2 Checkpoint Body
- `checkpoint_tx_digest_list (aka tx_digest_list): list<bytes[32]>`

制約:
- `checkpoint_tx_digest_list` は **MUST** `03-deterministic-execution.md` の `DET_ORDER_V1` に一致する。
- `tx_count` は **MUST** `len(checkpoint_tx_digest_list)` と一致する。
- `tx_digest_merkle_root` は **MUST** 6.3 の計算結果と一致する。
- `checkpoint_tx_digest_list` は **MUST** consensus 層で確定済みの順序を使い、実行層は再順序化してはならない。

### 6.3 tx_digest_merkle_root（Merkle-v1）
- 葉: `leaf_i = SHA3-256(tx_digest_i)`
- 内部ノード: `parent = SHA3-256(left || right)`
- 奇数葉の場合: **MUST** 末尾 leaf を自己複製してペア化
- 葉 0 件の場合: **MUST** `SHA3-256(empty bytes)`

### 6.4 checkpoint_digest
`checkpoint_digest = SHA3-256(MCS-1(CheckpointHeader))`

## 7. 妥当性ルール
- Header/Body が MCS-1 でない Checkpoint は **MUST** 無効。
- `prev_checkpoint_digest` 不一致 Checkpoint は **MUST** 無効。
- `validator_set_hash` 不一致 Checkpoint は **MUST** 無効。
- `tx_ordering_rule_id != 1` または `execution_rule_id != 1` は **MUST** 無効。

## 8. 擬似コード

### 8.1 Object 更新
```text
function apply_object_write(current_obj, op):
    assert current_obj.owner_kind != Immutable
    next_obj = op.apply(current_obj)
    next_obj.version = current_obj.version + 1
    return next_obj
```

### 8.2 Checkpoint Header 検証
```text
function validate_checkpoint_header(h, prev_digest, expected_vs_hash):
    assert h.checkpoint_format_version == 1
    assert h.prev_checkpoint_digest == prev_digest
    assert h.validator_set_hash == expected_vs_hash
    assert h.tx_ordering_rule_id == 1
    assert h.execution_rule_id == 1
```

## 9. 他仕様参照
- finality / epoch / validator set: `02-consensus.md`
- 決定論順序・競合解消: `03-deterministic-execution.md`
- state_root 算出対象ストレージ: `05-storage-layout.md`
