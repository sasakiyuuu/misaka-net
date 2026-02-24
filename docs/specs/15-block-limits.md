# 15 Block Limits 仕様

## 1. 目的と適用範囲
本仕様は block 単位の上限（bytes / tx count / gas）と reject 規則を定義する。
本仕様は `04-resource-limits.md` の TX/mempool 上限と整合して適用される。

## 2. 規範キーワード
- **MUST**: 準拠実装に必須
- **SHOULD**: 特段の理由がない限り必須
- **MAY**: 任意

## 3. 上限値（固定）
- `MAX_BLOCK_BYTES = 4 MiB`
- `MAX_TX_COUNT_PER_BLOCK = 2_000`
- `MAX_GAS_PER_BLOCK = 40_000_000`

## 4. 集計規則
- `block_bytes` は **MUST** `ordered_tx_list` の MCS-1 bytes 合計で計算。
- `tx_count` は **MUST** `len(ordered_tx_list)`。
- `gas_total` は **MUST** `sum(tx.gas_budget)`。

## 5. 順序規則整合
- `ordered_tx_list` は **MUST** consensus で確定した順序を使用。
- `checkpoint_tx_digest_list` は **MUST** `DET_ORDER_V1` に一致。
- 実行層は **MUST NOT** ブロック内順序を変更。

## 6. Reject 規則
- `block_bytes > MAX_BLOCK_BYTES` は **MUST** reject（`ERR_BLOCK_BYTES_EXCEEDED`）。
- `tx_count > MAX_TX_COUNT_PER_BLOCK` は **MUST** reject（`ERR_BLOCK_TX_COUNT_EXCEEDED`）。
- `gas_total > MAX_GAS_PER_BLOCK` は **MUST** reject（`ERR_BLOCK_GAS_EXCEEDED`）。
- いずれか違反時、その block は **MUST NOT** finality 対象に進む。

## 7. 提案者側事前検証
- block proposer は送信前に **MUST** 3上限を検証。
- 受信ノードは再検証を **MUST** 実施。

## 8. 8GB整合要件（worst-case stress model）
- 上限値は **MUST** `04-resource-limits.md` の 8GB envelope を前提に維持。
- 上限変更は **MUST** ガバナンス + timelock を経由。

### 8.1 定義
- `M_budget = 7.2 GiB`（`04-resource-limits.md`）
- `B = block_bytes`
- `T = tx_count`
- `S_tx = MAX_TX_SIZE = 128 KiB`（`04-resource-limits.md`）
- `P = MAX_GOSSIP_PARALLEL_RX = 16`
- `M_dedup = TX_DEDUP_CACHE_BYTES = 128 MiB`
- `VM_tmp_per_tx = 512 KiB`
- `f_shared = SHARED_TX_FRACTION = 0.60`
- `M_shared_per_tx = 96 KiB`

### 8.2 worst-case 式
- `M_block_ingest = B + (P * S_tx) + M_dedup`
- `M_vm_tmp = T * VM_tmp_per_tx`
- `M_shared = (T * f_shared) * M_shared_per_tx`
- `M_total_worst = M_block_ingest + M_vm_tmp + M_shared`

数値例（上限時）:
- `M_block_ingest = 4 MiB + (16 * 128 KiB) + 128 MiB = 134 MiB`
- `M_vm_tmp = 2_000 * 512 KiB = 1_000 MiB`
- `M_shared = (2_000 * 0.60) * 96 KiB = 112.5 MiB`
- `M_total_worst = 1_246.5 MiB`

実装は **MUST** 以下を満たす。
- `M_total_worst <= 1.8 GiB`（block 処理に許容する上限）
- かつ `04-resource-limits.md` の全体予算 `M_peak <= 7.2 GiB`

### 8.3 Shared TX 偏重ケース
- `shared_tx_count / tx_count > 0.70` の block は **MUST** reject（`ERR_BLOCK_SHARED_RATIO_EXCEEDED`）。
- `gas_total > 0.85 * MAX_GAS_PER_BLOCK` かつ `shared_tx_count / tx_count > 0.60` の block は **MUST** reject（`ERR_BLOCK_STRESS_ENVELOPE_EXCEEDED`）。

## 9. 監査ログ
- 上限違反 reject は **MUST** reason code とともに記録。
- ログは **MUST** `checkpoint_seq` 参照可能。
- stress model 判定値（`M_block_ingest`, `M_vm_tmp`, `M_shared`, `M_total_worst`）は **MUST** 記録。

## 10. 他仕様参照
- `02-consensus.md`
- `03-deterministic-execution.md`
- `04-resource-limits.md`
- `11-governance-and-emergency-mode.md`
- `14-p2p-networking.md`
