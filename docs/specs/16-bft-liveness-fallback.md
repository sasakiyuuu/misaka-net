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

## 9. 他仕様参照
- `02-consensus.md`
- `03-deterministic-execution.md`
- `08-state-commitment-and-snapshots.md`
- `09-weak-subjectivity.md`
- `11-governance-and-emergency-mode.md`
