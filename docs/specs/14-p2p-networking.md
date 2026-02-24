# 14 P2P Networking 仕様

## 1. 目的と適用範囲
本仕様は Misaka の P2P 層における handshake、peer 制限、gossip、peer scoring、DoS 防御を定義する。

## 2. 規範キーワード
- **MUST**: 準拠実装に必須
- **SHOULD**: 特段の理由がない限り必須
- **MAY**: 任意

## 3. Handshake

### 3.1 handshake フィールド（MCS-1順）
1. `p2p_format_version: u16`（固定値 `1`）
2. `chain_id: u32`
3. `protocol_version: u16`（固定値 `1`）
4. `node_id: bytes[32]`（Ed25519 公開鍵。handshake 署名検証キーとして使用）
5. `best_checkpoint_seq: u64`
6. `best_checkpoint_digest: bytes[32]`
7. `validator_set_hash: bytes[32]`
8. `listen_addr: string`
9. `timestamp_ms: u64`
10. `signature: bytes[64]`（Ed25519 固定長）

### 3.2 検証規則
- handshake 署名方式は **MUST** Ed25519 を使用する。
- `"MISAKA_P2P_HS_V1"` は **MUST** ASCII bytes literal として扱う。
- 署名対象は **MUST** `handshake_signing_message = SHA3-256("MISAKA_P2P_HS_V1" || MCS-1(fields 1..9))` とする。
- `signature` は **MUST** field 4 の `node_id`（Ed25519 公開鍵）で検証成功しなければならない。
- `chain_id` 不一致は **MUST** 切断。
- `protocol_version` 非互換は **MUST** 切断。
- `validator_set_hash` 検証不能は **MUST** 切断。
- handshake 署名不正は **MUST** 切断。

### 3.3 タイムアウト
- handshake 完了は **MUST** `T_HANDSHAKE_MAX = 5s` 以内。
- 超過は **MUST** 切断。

## 4. Peer Limits
- `MAX_INBOUND_PEERS = 48`
- `MAX_OUTBOUND_PEERS = 16`
- 同一 /24 サブネット上限は **MUST** `<= 2`
- 同一 ASN 上限は **MUST** inbound `<= 8`, outbound `<= 2`
- 同一 `node_id` の重複接続は **MUST NOT** 許可。

### 4.1 Eclipse 防御（必須）
- outbound peer 集合は **MUST** 少なくとも 8 個の異なる /24 を含む。
- outbound peer 集合は **MUST** 少なくとも 4 個の異なる ASN を含む。
- outbound peer の 25% は **MUST** `OUTBOUND_ROTATE_INTERVAL = 30 min` ごとに再接続する。
- inbound 受入は **MUST** ランダム化し、接続確立時刻の偏りを抑制する。

### 4.2 Bootstrap 戦略（必須）
- ノードは **MUST** `SEED_SET_MIN = 3` 個の独立 seed source を保持。
- 各 seed source からの初期接続数は **MUST** `<= 1/3 * MAX_OUTBOUND_PEERS`。
- bootstrap 直後の同期先は **MUST** `validator_set_hash` 一致 peer を優先。

## 5. Gossip

### 5.1 メッセージ種別
- `GossipTx`
- `GossipCheckpointHeader`
- `GossipCheckpointTxDigestList`

### 5.2 伝播規則
- 同一 `tx_hash` は **MUST** 再配布しない（de-dup）。
- `GossipCheckpointTxDigestList` は **MUST** `checkpoint_seq` を含む。
- `checkpoint_tx_digest_list` は **MUST** `DET_ORDER_V1` に一致。
- 検証不能メッセージは **MUST** 破棄。

## 6. Peer Scoring

### 6.1 スコア範囲
- `peer_score` は **MUST** `[0,100]`。
- 初期値は **MUST** `50`。

### 6.2 加点/減点
- 有効 checkpoint 伝播: `+1`
- 無効 TX 送信: `-5`
- 無効 checkpoint header: `-20`
- `checkpoint_tx_digest_list` 順序不一致: `-30`
- `validator_set_hash` 不一致を繰り返し送信: `-50`

### 6.3 追放
- `peer_score < 20` は **MUST** 切断。
- 連続違反 3 回で **MUST** `BAN_DURATION = 10 min`。

## 7. DoS 防御
- 受信 `GossipTx` は peer ごとに **MUST** `<= 100 tx/sec`。
- 受信 `GossipTx` サイズは **MUST** `04-resource-limits.md` の `MAX_TX_SIZE` 以下。
- block 関連受信は **MUST** `15-block-limits.md` 上限を超える payload を reject。
- メモリ圧迫時は **MUST** `04` の admission tightening に従う。

### 7.1 帯域上限（必須）
- peer あたり受信帯域は **MUST** `<= 1 MiB/s`。
- peer あたり送信帯域は **MUST** `<= 1 MiB/s`。
- ノード全体の受信帯域は **MUST** `<= 24 MiB/s`。
- ノード全体の送信帯域は **MUST** `<= 24 MiB/s`。

### 7.2 burst 制御（必須）
- 受信キュー総量は **MUST** `RX_QUEUE_MAX_BYTES = 64 MiB` 以下。
- peer 単位の burst は **MUST** `RX_BURST_MAX_BYTES = 4 MiB` 以下。
- 上限超過 peer は **MUST** 1 分間 throttling、3 回連続で **MUST** BAN。

## 8. 監査ログ
- 切断、BAN、重大不整合（順序不一致等）は **MUST** 監査ログ記録。
- ログは **MUST** `checkpoint_seq` と紐付け可能形式で保持。

## 9. 他仕様参照
- `01-tx-object-checkpoint.md`
- `02-consensus.md`
- `03-deterministic-execution.md`
- `04-resource-limits.md`
- `15-block-limits.md`
