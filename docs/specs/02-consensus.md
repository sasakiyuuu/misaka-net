# 02 Consensus 仕様（Phase 0/1）

## 1. 目的と適用範囲
本仕様は Misaka Phase 0/1 のコンセンサス層を定義する。
Phase 1 の採用方式は **CometBFT/Tendermint 系**（CometBFT v0.38 互換）とする。

本仕様は以下を固定する。
- finality 定義
- epoch と validator set 更新
- 実行層へ渡す順序保証

## 2. 規範キーワード
- **MUST**: 準拠実装に必須
- **SHOULD**: 特段の理由がない限り必須
- **MAY**: 任意

## 3. 前提モデル
- 1 ブロック = 1 提案 = BFT 合意対象。
- Checkpoint は **MUST** 1 つ以上の連続ブロックから構成可能。
- Phase 0/1 では Checkpoint 境界を **MUST** block height 境界に一致させる。
- consensus 層は **MUST** Checkpoint ごとに実行順序確定済み `checkpoint_tx_digest_list` を生成する。

## 4. Finality 定義

### 4.1 ブロック finality
ブロック `B(h)` は以下を満たした時点で final とする。
- CometBFT commit 証跡（`>2/3` voting power 署名）を持つ
- 直前 final ブロックとの連結性が検証済み

final ブロックは **MUST NOT** 巻き戻される。

### 4.2 Checkpoint finality
Checkpoint `C(s)` は、含有する全ブロックが final になった時点で final とする。
`C(s)` final 後、`state_root(C(s))` は **MUST NOT** 変更される。

## 5. Epoch 定義

### 5.1 Epoch 境界
- `EPOCH_LENGTH_BLOCKS`（固定値: `86,400` blocks）を **MUST** 使用。
- `height % EPOCH_LENGTH_BLOCKS == 0` の次ブロック高を新 epoch 開始点とする。

### 5.2 Validator set 適用タイミング
- Epoch `e` で確定した validator set 変更は **MUST** `e+1` 開始時に適用。
- 同 epoch 内で validator set は **MUST** 不変。

## 6. Validator Set 更新

### 6.1 入力
validator set 更新入力は以下を **MUST** 含む。
- `validator_pubkey: bytes[32]`
- `voting_power: u64`
- `activation_epoch: u64`
- `deactivation_epoch: u64?`（未指定可）

### 6.2 正当性
- `sum(voting_power)` は **MUST** `u64` 範囲内。
- 同一 `validator_pubkey` の重複 active は **MUST NOT** 許可。
- `activation_epoch` は **MUST** 現在 epoch 以上。

### 6.3 validator_set_hash
- active validator を `validator_pubkey` bytewise 昇順に並べる。
- 各要素を `validator_pubkey || little_endian_u64(voting_power)` で連結。
- `validator_set_hash = SHA3-256(concatenated_bytes)` を **MUST** 用いる。
- `01-tx-object-checkpoint.md` の `CheckpointHeader.validator_set_hash` は **MUST** これと一致。

## 7. 実行層インターフェース

### 7.1 consensus が提供するデータ
consensus 層は実行層に以下を **MUST** 提供する。
- `ordered_tx_list`（ブロック内順序保持）
- `checkpoint_tx_digest_list`（`03-deterministic-execution.md` の `DET_ORDER_V1` で正規化済み）
- `block_height`
- `epoch`
- `validator_set_hash`

### 7.2 禁止事項
- 実行層は **MUST NOT** `ordered_tx_list` のブロック内順序を変更してはならない。
- 実行層は **MUST** `checkpoint_tx_digest_list` をそのまま実行順序として使用し、再順序化してはならない。

## 8. 失敗時挙動
- commit 証跡不正ブロックは **MUST** reject。
- `validator_set_hash` 不一致ブロックは **MUST** reject。
- reject 後、ノードは **MUST** 最終 final 高さまでローカル状態を維持。

## 9. 擬似コード

### 9.1 block finality 判定
```text
function is_final_block(block, commit, validator_set):
    assert verify_commit_signatures(commit, validator_set)
    assert commit.voting_power_signed > (2/3 * validator_set.total_power)
    return true
```

### 9.2 epoch 適用
```text
function on_new_height(height):
    if height % EPOCH_LENGTH_BLOCKS == 1:
        validator_set = pending_set_for_epoch(current_epoch)
```

## 10. 他仕様参照
- TX/Object/Checkpoint 構造: `01-tx-object-checkpoint.md`
- 実行順序と競合解消: `03-deterministic-execution.md`
- リソース上限: `04-resource-limits.md`
