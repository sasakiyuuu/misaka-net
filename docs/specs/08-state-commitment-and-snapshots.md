# 08 State Commitment and Snapshots 仕様

## 1. 目的と適用範囲
本仕様は state root 算出、状態コミットメント、snapshot 構成と検証手順を定義する。
Phase 1 の StateRoot-v1（`05-storage-layout.md`）との整合を保ちながら、Jellyfish Merkle（JMT）移行を規定する。

## 2. 規範キーワード
- **MUST**: 準拠実装に必須
- **SHOULD**: 特段の理由がない限り必須
- **MAY**: 任意

## 3. State Root ハッシュ対象（固定）
state root 算出対象は **MUST** 以下のみ。
1. `object_store`
2. `version_index_latest_only`
3. `sys/params/*`

除外対象（**MUST NOT**）:
- `event_log_store_recent`
- indexer 専用ストア（`owner_index` 等）

この対象定義は `05-storage-layout.md` と一致しなければならない。

## 4. StateRoot バージョン
- `StateRoot-v1`: `05-storage-layout.md` の規則
- `StateRoot-v2`: 本仕様の JMT 規則

`sys/params/state_root_version` は **MUST** `1` または `2`。

## 5. Jellyfish Merkle（StateRoot-v2）
### 5.1 キー正規化
- すべての key は **MUST** バイト列として扱う。
- `key` 比較規則は **MUST** bytewise lexicographic。
- JMT パスキーは **MUST** `path = SHA3-256(key_bytes)`。

### 5.2 Leaf ハッシュ
`leaf_hash = SHA3-256(0x00 || u32le(len(key)) || key || u32le(len(value)) || value)`

### 5.3 Internal ハッシュ
`node_hash = SHA3-256(0x01 || child_0 || child_1 || ... || child_15)`

### 5.4 空ノード / 空状態
- 空ノード hash は **MUST** `ZERO_HASH = 32-byte zero`。
- 子を持たない internal nibble は **MUST** `ZERO_HASH` を使って埋める。
- エントリ 0 件の空状態 root は **MUST** `ZERO_HASH` とする（v1 の `SHA3-256(empty bytes)` とは別定義）。

### 5.5 挿入アルゴリズム（必須）
- 入力は **MUST** `path`（32 bytes）昇順で処理する。
- 同一 `path` が複数存在する入力は **MUST** reject（`ERR_STATE_ROOT_DUPLICATE_PATH`）。
- 既存 leaf と prefix 衝突した場合、**MUST** 最長共通 prefix 深度まで internal node を展開し分岐する。
- `key` 順と `path` 順が異なる場合でも、挿入順は **MUST** `path` 順を優先する。

### 5.6 root 算出手順（必須）
1. `kv_entries` を `path = SHA3-256(key)` 昇順で正規化（同値時は `key` 昇順）。
2. 各 entry から `path` と `leaf_hash` を算出。
3. JMT に正規化済み順（path 順）で leaf を挿入。
4. bottom-up に `node_hash` を再計算。
5. ルート hash を `state_root_v2` とする。

### 5.7 決定性要件
同一 key/value 集合に対し、全実装は **MUST** 同一 `state_root` を生成する。

## 6. v1→v2 移行規則
- 移行 checkpoint `U` を定義する。
- `checkpoint_seq < U` は **MUST** v1。
- `checkpoint_seq >= U` は **MUST** v2。
- `U` で以下を **MUST** 検証。
  1. v1 root の再計算一致
  2. v2 root の再計算一致
  3. `sys/params/state_root_version == 2`

不一致時は **MUST** Checkpoint reject。

## 7. Snapshot 最小構成
snapshot は **MUST** 以下を含む。
1. `SnapshotHeader`
   - `chain_id`
   - `checkpoint_seq`
   - `checkpoint_digest`
   - `state_root`
   - `state_root_version`
   - `validator_set_hash`
2. `object_store`（最新）
3. `version_index_latest_only`
4. `sys/params/*`

## 8. Snapshot 検証手順
ノードは復元時に **MUST** 以下を実行。
1. `checkpoint_digest == SHA3-256(MCS-1(CheckpointHeader))`
2. `validator_set_hash` の一致確認
3. `state_root_version` に応じた root 再計算
4. `snapshot_checkpoint + 1` から replay
5. replay 後 root 一致確認

いずれか不一致なら **MUST** 起動拒否。

## 9. 擬似コード
```text
function compute_state_root_v2_jmt(kv_entries):
    if len(kv_entries) == 0:
        return ZERO_HASH

    sorted = sort_by_path_then_key(kv_entries)  // path=SHA3_256(key), tie-breaker=key
    paths = []
    for (k,v) in sorted:
        p = SHA3_256(k)
        if exists(paths, p):
            reject(ERR_STATE_ROOT_DUPLICATE_PATH)
        paths.append(p)
        insert_leaf_jmt(p, SHA3_256(0x00 || u32le(len(k)) || k || u32le(len(v)) || v))

    return recompute_root_bottom_up_with_zero_hash_padding()

function verify_and_restore(snapshot):
    assert hash_checkpoint_header(snapshot.header.checkpoint_header) == snapshot.header.checkpoint_digest
    assert verify_validator_set_hash(snapshot.header.validator_set_hash)

    if snapshot.header.state_root_version == 1:
        root = compute_state_root_v1(snapshot.kv)
    else:
        root = compute_state_root_v2_jmt(snapshot.kv)

    assert root == snapshot.header.state_root

    state = load_snapshot(snapshot)
    state = replay_from(snapshot.header.checkpoint_seq + 1, state)
    assert compute_state_root(state) == expected_root_after_replay
    return state
```

## 10. テストベクタ要件
### 10.1 必須ベクタ
- 同一状態を異なるノード実装に入力し、同一 root を得るケース
- tombstone を含むケース
- `sys/params/*` 変更を含むケース

### 10.2 形式
- `state_root_version`
- `kv_entries`
- `expected_state_root`

詳細形式と必須ケース定義は **MUST** `13-test-vectors.md` に従う。

## 11. 他仕様参照
- `01-tx-object-checkpoint.md`
- `05-storage-layout.md`
- `09-weak-subjectivity.md`
- `13-test-vectors.md`
