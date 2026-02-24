# 04 Resource Limits 仕様（8GB Validator Safety Envelope）

## 1. 目的と適用範囲
本仕様は 8GB RAM バリデータを現実的に運用するための上限値と根拠を定義する。
DoS 耐性のため、全上限は数値固定する。

## 2. 規範キーワード
- **MUST**: 準拠実装に必須
- **SHOULD**: 特段の理由がない限り必須
- **MAY**: 任意

## 3. メモリ予算モデル（8GiB）

### 3.1 総予算
- 物理 RAM 想定: `M_total = 8 GiB`
- OOM 回避のため、実効上限は **MUST** `M_budget = 7.2 GiB`（90%）

### 3.2 コンポーネント予算（上限）
- `M_os_runtime = 1.0 GiB`（OS/ランタイム/ログ）
- `M_consensus = 1.0 GiB`（CometBFT internal + P2P vote buffers）
- `M_rocksdb_block_cache = 2.5 GiB`
- `M_vm_cache = 1.2 GiB`（Move VM code/data cache）
- `M_mempool = 1.0 GiB`
- `M_margin = 0.5 GiB`（突発バースト吸収）

合計:
`M_peak = M_os_runtime + M_consensus + M_rocksdb_block_cache + M_vm_cache + M_mempool + M_margin = 7.2 GiB`

実装は **MUST** `M_peak <= M_budget` を維持する。

### 3.3 受け入れ基準（メモリ）
- ノード設定の静的検査で **MUST** `M_peak <= 7.2 GiB` を満たす。
- 実行時観測で `rss_process > 7.2 GiB` が 10 秒継続した場合、ノードは **MUST** admission tightening を開始する。

### 3.4 admission tightening（定義）
`rss_process > 7.2 GiB` が 10 秒継続した場合、ノードは次を **MUST** 実施。
1. 新規 mempool 受入サイズ上限を一時的に `MAX_TX_SIZE=64 KiB` に引き下げ
2. sender ごとの保有上限を一時的に `64` に引き下げ
3. `MAX_MEMPOOL_BYTES` 超過分を低優先度から削除

通常モード復帰条件:
- `rss_process <= 6.8 GiB` が 60 秒継続

## 4. TX サイズ/複雑度上限

### 4.1 上限値（固定）
- `MAX_TX_SIZE = 128 KiB`（MCS-1 エンコード後サイズ）
- `MAX_OBJECT_SIZE = 64 KiB`（Object.payload の MCS-1 エンコード後サイズ）
- `MAX_WRITES_PER_TX = 32 objects`
- `MAX_INPUTS_PER_TX = 64 objects`
- `MAX_EVENTS_PER_TX = 128`
- `MAX_CALL_DEPTH = 32`
- `MAX_GAS_BUDGET = 50_000_000`

### 4.2 根拠（計測前提/保守値）
- 8GB では、1 TX の大量 write が VM 一時メモリを押し上げる。
- `MAX_WRITES_PER_TX=32` により payload 上限は概ね `32 * 64KiB = 2MiB`。
- `MAX_TX_SIZE=128KiB` はネットワーク増幅と mempool 占有を抑える保守値。
- `MAX_CALL_DEPTH=32` は深い再帰呼び出し型 DoS を抑制。

実装は **MUST** 上限違反 TX を事前 reject する。

## 5. Mempool 上限
- `MAX_MEMPOOL_TX_COUNT = 20_000`
- `MAX_MEMPOOL_BYTES = 1 GiB`

受入規則:
- `MAX_MEMPOOL_BYTES` 到達時、ノードは **MUST** 低優先度 TX を追い出す。
- 同一 sender あたり保有数は **MUST** `<= 128`。

優先度定義:
- 優先度は **MUST** `effective_fee_per_byte = (gas_budget * gas_price) / tx_size_bytes` の降順。
- 同率時は **MUST** `received_at` 昇順（先着優先）。

## 6. ガスと実行時間の安全弁
- 各 TX は **MUST** `gas_budget` を超えて実行されない。
- ガス不足時は **MUST** 即時中断し、状態変更を適用しない。

## 7. 失敗時挙動
- 上限違反 TX は **MUST** consensus 提案前に reject。
- 上限違反を含むブロックは **MUST** 実行時 reject。
- reject 理由コードは **MUST** 以下を使用。
  - `ERR_TX_TOO_LARGE`
  - `ERR_OBJECT_TOO_LARGE`
  - `ERR_TOO_MANY_WRITES`
  - `ERR_TOO_MANY_INPUTS`
  - `ERR_CALL_DEPTH_EXCEEDED`
  - `ERR_GAS_BUDGET_EXCEEDED`

## 8. 8GB 想定の上限超過ケース（明示）
以下は **MUST** 警戒ケースとして実装に含める。
1. mempool 急増で `MAX_MEMPOOL_BYTES` 到達
2. 大型 TX 連打により VM cache + mempool が同時肥大
3. compaction と checkpoint 作成が重なり RocksDB cache がピーク

## 9. 他仕様参照
- 実行決定性: `03-deterministic-execution.md`
- ストレージ配置: `05-storage-layout.md`
- block 上限: `15-block-limits.md`
- P2P DoS 制御: `14-p2p-networking.md`
