# 02 Consensus 仕様（Phase 0/1）

## 1. 目的と適用範囲
本仕様は Misaka Phase 0/1 のコンセンサス層を定義する。
Phase 1 の採用方式は **CometBFT/Tendermint 系**（CometBFT v0.38 互換）とする。

### 1.1 Phase 1 の位置づけ（戦略明示）
- Phase 1 は **MUST** CometBFT 依存の現実運用フェーズとして扱う。
- Phase 1 は **MUST NOT** 「独立実装済み L1 と同等の分散性能」を主張しない。
- 独立 L1 への移行は **MAY** 検討するが、Phase 1 仕様準拠条件には含めない。

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

### 4.3 `finality_proof_v1`（trusted checkpoint 用）
`09-weak-subjectivity.md` で使用する finality 証跡は **MUST** 次形式とする。

構造体（MCS-1 順）:
1. `proof_format_version: u16`（固定値 `1`）
2. `chain_id: u32`
3. `checkpoint_seq: u64`
4. `checkpoint_digest: bytes[32]`
5. `finalized_block_height: u64`
6. `finalized_round: u32`
7. `finalized_block_id: bytes[32]`
8. `validator_set_hash: bytes[32]`
9. `validator_set_entries: list<ValidatorPowerEntry>`
10. `commit_signatures: list<CommitSig>`

`ValidatorPowerEntry`（MCS-1 順）:
1. `validator_pubkey: bytes[32]`
2. `voting_power: u64`

`CommitSig`（MCS-1 順）:
1. `validator_pubkey: bytes[32]`
2. `signature: bytes[64]`（Ed25519）

署名対象:
- `commit_signing_message = CometBFTCanonicalVoteSignBytes(chain_id, finalized_block_height, finalized_round, finalized_block_id, vote_type=PRECOMMIT)` を **MUST** 使用する。

検証規則:
- `proof_format_version == 1` を **MUST** 満たす。
- `checkpoint_digest` は **MUST** `01-tx-object-checkpoint.md` の `CheckpointHeader` と一致する。
- `finalized_block_height` は **MUST** `checkpoint_seq` に対応する final 済みブロック列に含まれる。
- `validator_set_entries` は **MUST** `validator_pubkey` bytewise 昇順かつ重複なし。
- `validator_set_hash` は **MUST** `SHA3-256(concat(validator_pubkey || u64_le(voting_power)))`（昇順連結）と一致する。
- 各 `commit_signatures` 要素は **MUST** `validator_set_entries` 内の `validator_pubkey` と一致し、`commit_signing_message` に対して署名検証成功する。
- `commit_signatures` は **MUST** `validator_pubkey` 重複を含まない。
- 有効署名 voting power 合計は **MUST** `> 2/3`。

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

### 6.3 最小 validator 集合（必須）
- `MIN_VALIDATOR_SET_SIZE = 4`（初期値）を **MUST** 使用する。
- finality 判定対象の active validator 数は **MUST** `>= MIN_VALIDATOR_SET_SIZE`。
- active validator 数が 0 の集合は **MUST NOT** 生成/適用する（`ERR_EMPTY_VALIDATOR_SET`）。
- `sum(voting_power)` が 0 の集合は **MUST NOT** 生成/適用する（`ERR_ZERO_TOTAL_VOTING_POWER`）。
- `MIN_VALIDATOR_SET_SIZE` 違反の更新提案は **MUST** `ERR_VALIDATOR_SET_TOO_SMALL` で reject。

### 6.4 validator_set_hash
- active validator を `validator_pubkey` bytewise 昇順に並べる。
- 各要素を `validator_pubkey || little_endian_u64(voting_power)` で連結。
- `validator_set_hash = SHA3-256(concatenated_bytes)` を **MUST** 用いる。
- `01-tx-object-checkpoint.md` の `CheckpointHeader.validator_set_hash` は **MUST** これと一致。

### 6.5 境界条件（必須）
- epoch 境界で pending set を適用する際、`MIN_VALIDATOR_SET_SIZE` / total voting power 条件を再検証し、違反時は **MUST** 現行 set を維持して fail-close する。
- fail-close 発生時は **MUST** 監査ログへ記録する。
### 6.6 決定論時刻 `now_ms`（必須）
- 本仕様群で使用する `now_ms` は **MUST** `CheckpointHeader.timestamp_ms`（`01-tx-object-checkpoint.md`）を指す。
- `now_ms` は wall-clock を直接参照してはならず、**MUST NOT** ノードローカル時刻から計算する。
- `checkpoint_seq=s` に対する `now_ms` は **MUST** `C(s).timestamp_ms` と一致する。

## 7. 実行層インターフェース

### 7.1 consensus が提供するデータ
consensus 層は実行層に以下を **MUST** 提供する。
- `ordered_tx_list`（ブロック内順序保持）
- `checkpoint_tx_digest_list`（`03-deterministic-execution.md` の `DET_ORDER_V1` で正規化済み）
- `block_height`
- `epoch`
- `validator_set_hash`

生成責務:
- consensus 層は **MUST** checkpoint 対象 TX 集合に `DET_ORDER_V1` を適用して `checkpoint_tx_digest_list` を生成する。
- `checkpoint_tx_digest_list` が `DET_ORDER_V1` 検証に失敗する場合、その checkpoint は **MUST** reject（`ERR_DET_ORDER_MISMATCH`）。

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

### 9.3 Adversarial Simulation / Fork Proof 要件（必須）
準拠実装は **MUST** 以下を実施する。
1. 同一 `checkpoint_seq` に対し異なる `finalized_block_id` を含む `finality_proof_v1` を入力し、両方 reject を確認。
2. 旧 `validator_set_hash` を混入した `finality_proof_v1` を入力し reject を確認。
3. commit 署名集合が `>2/3` を満たさない fork 証跡を入力し reject を確認。

拒否時エラーコード:
- `ERR_FORK_FINALITY_CONFLICT`
- `ERR_FORK_VALIDATOR_SET_MISMATCH`
- `ERR_FORK_INSUFFICIENT_COMMIT_POWER`

上記テスト入力/期待結果は **MUST** `13-test-vectors.md` 形式で保存する。

## 10. 他仕様参照
- TX/Object/Checkpoint 構造: `01-tx-object-checkpoint.md`
- 実行順序と競合解消: `03-deterministic-execution.md`
- リソース上限: `04-resource-limits.md`
- 決定性テストベクタ: `13-test-vectors.md`
- P2P handshake / peer 制御: `14-p2p-networking.md`
- block 上限: `15-block-limits.md`
