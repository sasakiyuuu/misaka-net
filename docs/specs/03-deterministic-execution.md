# 03 Deterministic Execution 仕様（Phase 0/1）

## 1. 目的と適用範囲
本仕様は、同一入力に対して全ノードが同一 `state_root` を生成する実行規則を定義する。
対象は以下。
- TX スケジューリング
- 競合解消
- shared object TX の完全順序

## 2. 規範キーワード
- **MUST**: 準拠実装に必須
- **SHOULD**: 特段の理由がない限り必須
- **MAY**: 任意

## 3. 受け入れ基準（必須）
1. 同一 `pre_state_root` と同一 `checkpoint_tx_digest_list` を入力した全ノードは、**MUST** 同一 `state_root` を出力する。
2. shared object を含む TX の順序は **MUST** 一意に決まり、実装依存の余地を持たない。
3. 競合シナリオ（11.1, 11.2）で成功/失敗 TX セットと最終 `state_root` は **MUST** 一致する。

## 4. 決定論の前提
決定論は次が一致する場合に成立しなければならない。
1. 初期 state（`checkpoint_seq = s-1` の `state_root`）
2. final 済み Checkpoint `C(s)` の `checkpoint_tx_digest_list`
3. 本仕様の順序規則

上記が一致する場合、実装は **MUST** 同一 `state_root(C(s))` を生成する。

## 5. TX 分類
- **Owned-only TX**: Shared input を含まない TX
- **Shared TX**: 1 つ以上の Shared input を含む TX
- **Mixed TX**: Shared input と Owned input を併せ持つ TX（Shared TX の一種）

## 6. 実行前正規化
各 TX について以下を **MUST** 計算する。
- `tx_hash`
- `shared_input_ids`（Shared input の object_id を bytewise lexicographic 昇順で並べた配列）
- `primary_shared_id`（`shared_input_ids` の先頭。空なら `None`）

`None` は内部表現のみであり、外部シリアライズ対象ではない。

## 7. 完全順序規則（DET_ORDER_V1）

### 7.1 グローバル原則
1. Checkpoint 内順序は **MUST** 一意。
2. 任意 2 TX について「先/後」が **MUST** 決定可能。

### 7.2 Owned-only TX 順序
Owned-only TX 群は **MUST** `tx_hash` bytewise 昇順で順序化する。

### 7.3 Shared TX 順序
Shared TX 群は **MUST** 次のキーで辞書順に並べる。
1. `primary_shared_id` bytewise 昇順
2. `tx_hash` bytewise 昇順

### 7.4 最終 merge ルール
Checkpoint 内の最終実行順序は **MUST** 以下で決定する。
1. Shared TX 群（7.3 の順）
2. Owned-only TX 群（7.2 の順）

注記:
- Mixed TX は Shared TX 群に含むため、Owned-only より常に先行。
- `02-consensus.md` の `checkpoint_tx_digest_list` はこの順序（DET_ORDER_V1）で確定済みとする。
- 実行層は **MUST** `checkpoint_tx_digest_list` をそのまま実行し、再順序化してはならない。
- 実行層は **MUST** 実行前に `checkpoint_tx_digest_list` が DET_ORDER_V1 に一致することを検証し、不一致なら Checkpoint を reject する（`ERR_DET_ORDER_MISMATCH`）。

### 7.5 責務分離（必須）
- **consensus 層**は **MUST** `ordered_tx_list` から `DET_ORDER_V1` を適用して `checkpoint_tx_digest_list` を生成・最終確定する。
- **execution 層**は **MUST NOT** 順序生成責務を持たず、受領した `checkpoint_tx_digest_list` の検証と実行のみを行う。
- consensus と execution の双方で `DET_ORDER_V1` 検証結果が不一致の場合、当該 checkpoint は **MUST** reject。

## 8. ロック取得順

### 8.1 ロックキー
TX は mutable input に対し排他ロックを取得する。ロックキーは `object_id`。

### 8.2 取得順
- 1 TX 内ロック取得順は **MUST** `object_id` bytewise 昇順。
- ロック解放順は **MUST** 取得逆順。

## 9. 競合解消と失敗時ガス（必須）

### 9.1 同一 Owned object 競合
- 先行 TX が version を進めた後、後続 TX の `expected_version` 不一致は **MUST** 失敗。
- 失敗 TX は **MUST** アクション由来の状態変更を行わない（ガス課金のみ実施）。

### 9.2 Shared object 混在競合
- Shared TX は 7.3 に従って逐次適用。
- 後続 TX は直前反映済み shared object の最新版を読む。
- 前提条件不一致は **MUST** 失敗し、アクション由来の状態変更を行わない。

### 9.3 失敗 TX の課金セマンティクス（必須）
- 失敗 TX でも **MUST** `gas_used` を算出し、`gas_charged = min(gas_used, gas_budget)` を課金する。
- `charge_gas_only` は **MUST** 以下のみを変更可能とする。
  1. ガス支払いオブジェクト残高更新（`-fee_charged`、必要時 `+fee_refund`）
  2. fee 受取先の増加（`fee_charged`、protocol fee vault など）
- 上記以外の状態（`object_store`, `version_index` の一般オブジェクト更新、任意イベント）は **MUST NOT** 変更。
- `charge_gas_only` による残高更新は **MUST** `state_root` 算出対象に含まれる（`05-storage-layout.md`）。

## 10. 実行アルゴリズム擬似コード（DET_EXEC_V1）

```text
function execute_checkpoint(checkpoint_tx_digest_list, pre_state):
    assert verify_det_order_v1(checkpoint_tx_digest_list)

    ordered = resolve_tx_bodies(checkpoint_tx_digest_list)

    state = pre_state
    for tx in ordered:
        lock_ids = mutable_input_ids(tx)
        sort(lock_ids)  // bytewise lexicographic
        acquire_exclusive_locks(lock_ids)

        result = execute_tx(tx, state)
        if result.success:
            state = apply_effects(state, result.effects)
        else:
            state = apply_gas_charges_only(state, tx, result)

        release_locks_reverse(lock_ids)

    return compute_state_root(state)
```

## 11. 決定性検証シナリオ（必須）

### 11.1 シナリオA: 同一 object 競合
初期状態:
- `O1.version = 7`

TX:
- `T1`: `O1 expected_version=7` を更新
- `T2`: `O1 expected_version=7` を更新

手順:
1. `tx_hash(T1) < tx_hash(T2)` の場合、Owned-only 順序で `T1 -> T2`
2. `T1` 成功で `O1.version=8`
3. `T2` は version mismatch で失敗

要件:
- 全ノードで **MUST** `T1 success, T2 fail`
- 最終 `state_root` は **MUST** 一意

### 11.2 シナリオB: shared object 混在
初期状態:
- Shared `S1.version=10`
- Owned `A1.version=3`

TX:
- `T3`: `S1` mutable
- `T4`: `S1` mutable + `A1` mutable（Mixed）
- `T5`: `A1` mutable（Owned-only）

手順:
1. `T3,T4` は Shared 群、`T5` は Owned-only 群
2. Shared 順序は `(primary_shared_id, tx_hash)` で確定
3. Shared 群を先に全適用、その後 `T5`

要件:
- 全ノードで **MUST** 同一順序を選ぶ
- `T4` の `A1` 更新は Shared 群フェーズで反映される
- `T5` 成功/失敗は一意
- 最終 `state_root` は一意

## 12. 禁止事項
- 実装は **MUST NOT** ローカル時刻・スレッド順・乱数で順序決定してはならない。
- 実装は **MUST NOT** 失敗 TX でアクション由来の状態変更を行ってはならない（9.3 のガス課金状態更新を除く）。

## 13. 他仕様参照
- データ構造/Checkpoint: `01-tx-object-checkpoint.md`
- finality/epoch: `02-consensus.md`
- state_root 算出対象: `05-storage-layout.md`
- 決定性テストベクタ: `13-test-vectors.md`
- MEV方針: `17-mev-policy.md`
