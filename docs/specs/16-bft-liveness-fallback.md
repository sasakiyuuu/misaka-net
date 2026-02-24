# 16 BFT Liveness Fallback 仕様

## 1. 目的と適用範囲
本仕様は BFT liveness 停止時の検知・安全移行・復旧手順を定義する。
対象は consensus 停滞時の safe mode と通常運用復帰条件。

## 2. 規範キーワード
- **MUST**: 準拠実装に必須
- **SHOULD**: 特段の理由がない限り必須
- **MAY**: 任意

## 3. liveness 停止検知
以下のいずれか成立で停止判定。
1. final checkpoint 進行停止が `T_LIVENESS_STOP = 30 min` 超
2. block height 進行停止が `K_HEIGHT_STOP = 120 blocks` 超
3. commit 証跡収集失敗が `K_COMMIT_FAIL = 64` 連続

成立時、ノードは **MUST** safe mode へ遷移。

## 4. Safe Mode
safe mode 中、ノードは以下を **MUST** 実施。
- 新規 block proposal 停止
- 新規 `checkpoint_tx_digest_list` 生成停止
- register/unbond/slash の実行拒否
- read-only API は維持

safe mode 中は **MUST** `11-governance-and-emergency-mode.md` の fallback 状態を参照する。

### 4.1 経済処理連携（必須）
- safe mode 中は **MUST** `10-tokenomics.md` の safe mode 式を適用する。
- safe mode 中は **MUST** epoch 報酬配分を停止（`Reward_i = 0`）。
- safe mode 中は **MUST** grant 配分を停止（`GrantPool_t = 0`）。

### 4.2 long-stall 判定
- safe mode 継続が `T_STALL_LONG = 7 days` 以上で `long_stall_active=true`。
- long-stall 中のインフレ率は **MUST** `r_epoch_safe` を使用する（初期値 `0`）。

### 4.3 outage tier（必須）
- `T_STALL >= 7 days` で Tier-1（long-stall）
- `T_STALL >= 30 days` で Tier-2（governance freeze）
- `T_STALL >= 90 days` で Tier-3（tokenomics halt）

Tier 動作:
- Tier-1: `10-tokenomics.md` の safe mode 経済式を適用
- Tier-2: `sys/params/*` の恒久パラメータ更新を **MUST NOT** 実行
- Tier-3: `Inflation_t`, `Reward_i`, `GrantPool_t` を **MUST** 0 に固定

## 5. 復旧手順
1. 直近 final checkpoint を取得
2. `validator_set_hash` 一致を検証
3. `state_root` 再計算一致を検証
4. `checkpoint_seq + 1` から再同期
5. 連続進行確認

上記は **MUST** 順序どおり実施。

## 6. 復帰条件
以下すべてを満たす場合のみ safe mode を解除。
1. `K_RECOVER_FINAL = 10` 連続 final checkpoint 進行
2. `state_root` 不一致ゼロ
3. `validator_set_hash` 不一致ゼロ
4. fallback mode が非アクティブ

## 7. fallback mode 連携
- fallback mode が active の場合、safe mode 自動解除は **MUST NOT**。
- fallback 解除後、6章条件を満たしたときのみ **MUST** 通常復帰。

## 8. 監査ログ
- 停止検知、safe mode 遷移、復旧失敗、復帰判定は **MUST** 監査ログに記録。
- ログは **MUST** `checkpoint_seq` 参照可能。

### 8.1 Adversarial Simulation（必須）
- 実装は **MUST** 以下を定期実行し、結果を監査ログ化する。
  1. `K_COMMIT_FAIL` 連続失敗から safe mode 遷移
  2. forked finality 証跡入力（`02-consensus.md` の fork 要件）
  3. 7d/30d/90d 継続停止シナリオ
- 各シナリオで期待どおり Tier 遷移しない場合、ノードは **MUST** 通常復帰を拒否。

## 9. 他仕様参照
- `02-consensus.md`
- `03-deterministic-execution.md`
- `08-state-commitment-and-snapshots.md`
- `09-weak-subjectivity.md`
- `10-tokenomics.md`
- `11-governance-and-emergency-mode.md`
