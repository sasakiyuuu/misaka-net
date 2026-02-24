# 05 Storage Layout 仕様（Validator / Indexer 分離）

## 1. 目的と適用範囲
本仕様は以下を定義する。
- validator が合意に必要な最小ストレージのみを保持すること
- indexer 専用インデックスを validator から分離すること
- state_root 算出の決定性を固定すること

## 2. 規範キーワード
- **MUST**: 準拠実装に必須
- **SHOULD**: 特段の理由がない限り必須
- **MAY**: 任意

## 3. ストア分類

### 3.1 Validator 必須ストア
validator は以下を **MUST** 搭載する。
1. `object_store`
2. `version_index`
3. `checkpoint_store`
4. `event_log_store_recent`

### 3.2 Validator 非搭載（Indexer 専用）
validator は以下を **MUST NOT** 搭載する。
1. `owner_index`
2. 任意の全文検索インデックス
3. 任意の履歴集計マテリアライズドビュー

indexer は **MAY** 上記を搭載する。

## 4. キー仕様（エンコード固定）

### 4.1 共通
- 文字列プレフィックスは **MUST** ASCII bytes でエンコード。
- `object_id` は **MUST** 32 bytes。
- `version` は **MUST** `u64 little-endian`。

### 4.2 object_store
- key: `b"obj/" || object_id`
- value: 最新 Object 本体（MCS-1 bytes）

### 4.3 version_index
- key: `b"ver/" || object_id || b"/" || u64_le(version)`
- value: `object_digest(32 bytes) || lifecycle_state(u8) || checkpoint_seq(u64_le)`

### 4.4 checkpoint_store
- key: `b"ckpt/" || u64_le(checkpoint_seq)`
- value: `CheckpointHeader || tx_digest_list`

### 4.5 event_log_store_recent
- key: `b"evt/" || u64_le(checkpoint_seq) || b"/" || u32_le(tx_index) || b"/" || u32_le(event_index)`
- value: event bytes（MCS-1）

保持期間:
- `event_log_store_recent` は **MUST** 直近 `EVENT_RETENTION_CHECKPOINTS = 10_000` まで保持。
- 超過分は **MUST** pruning 対象。

## 5. Pruning / Snapshot 境界

### 5.1 pruning window
- validator の履歴保持窓は **MUST** `PRUNING_WINDOW_CHECKPOINTS = 50_000`。

### 5.2 snapshot 境界
- snapshot は **MUST** checkpoint 境界でのみ作成。
- snapshot は少なくとも以下を **MUST** 含む。
  - 全 `object_store` 最新値
  - `version_index`（window 内）
  - 最新 final checkpoint header
  - `validator_set_hash`

### 5.3 復元要件
- ノードは **MUST** snapshot 復元後、`snapshot_checkpoint + 1` から replay し、同一 `state_root` を得る。

## 6. State Root 算出対象
`state_root` 算出時、対象は **MUST** 以下のみ。
1. `object_store`
2. `version_index_latest_only`
3. `sys/params/*`

定義:
- `version_index_latest_only` は、各 `object_id` について `version` 最大の 1 エントリ（Tombstone を含む）。
- `sys/params/*` は固定キー集合の KV ストアとし、キー/値は MCS-1 bytes でエンコードする。
- Phase 4 で最低限保持するキー: `state_root_version`, `weak_subjectivity_period_ms`, `committee_n`, `committee_m`, `committee_pubkeys`, `committee_id`, `r_annual`, `p_validator`, `p_grants`, `p_treasury`, `timelock_checkpoints`。
- `sys/params/committee_pubkeys` は **MUST** `list<bytes[32]>`（bytewise 昇順、重複なし）とする。
- `len(sys/params/committee_pubkeys)` は **MUST** `committee_n` と一致する。
- `sys/params/committee_id` は **MUST** `SHA3-256(concat(sys/params/committee_pubkeys))` で算出する。

以下は **MUST NOT** ハッシュ対象。
- `event_log_store_recent`
- indexer 専用ストア（`owner_index` 等）

## 7. ハッシュ順序規則（StateRoot-v1）
- ハッシュ関数は **MUST** `SHA3-256`。
- `len(x)` は **MUST** `u32 little-endian`。
- leaf hash は **MUST** `SHA3-256(len(key)||key||len(value)||value)`。
- 対象エントリは全ストア横断で 1 つの配列に集約後、**MUST** key の bytewise lexicographic で単一ソート。
- Merkle 内部ノードは **MUST** `SHA3-256(left || right)`。
- 葉が奇数の場合、**MUST** 末尾 leaf を自己複製してペア化。
- 葉 0 件の場合、**MUST** `SHA3-256(empty bytes)`。
- 同一の入力エントリ集合（同一 key/value 集合）に対し、実装は **MUST** 同一 `state_root` を生成する。

## 8. 擬似コード
```text
function compute_state_root(stores):
    pairs = []

    for (k,v) in iterate_all(object_store):
        pairs.append((k,v))

    latest = pick_latest_version_per_object(version_index)  // include tombstone
    for (k,v) in iterate_all(latest):
        pairs.append((k,v))

    for (k,v) in iterate_all(sys_params_store):  // sys/params/* store
        pairs.append((k,v))

    pairs = sort_by_key_bytewise_lex(pairs)

    leaves = []
    for (k,v) in pairs:
        leaves.append(SHA3_256(u32le(len(k)) || k || u32le(len(v)) || v))

    return merkle_v1(leaves)
```

## 9. 8GB との整合要件
- validator は **MUST** `owner_index` を保持しない。
- RocksDB cache 設定は **MUST** `04-resource-limits.md` の予算を超えない。

## 10. 他仕様参照
- データ構造: `01-tx-object-checkpoint.md`
- 実行順序: `03-deterministic-execution.md`
- リソース上限: `04-resource-limits.md`
