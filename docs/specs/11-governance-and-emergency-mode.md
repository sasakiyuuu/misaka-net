# 11 Governance and Emergency Mode 仕様（Phase 4）

## 1. 目的と適用範囲
本仕様は以下を定義する。
- パラメータ更新権限と手続き
- timelock
- Solana 障害時 fallback mode
- emergency local override

## 2. 規範キーワード
- **MUST**: 準拠実装に必須
- **SHOULD**: 特段の理由がない限り必須
- **MAY**: 任意

## 3. ロール定義
- `GovCouncil`: 通常ガバナンス決定主体
- `EmergencyCouncil`: 緊急モードの発動/解除主体
- `ValidatorSet`: 最終承認と on-chain 反映主体

## 4. 権限分離
- `GovCouncil` は **MUST** 通常パラメータ変更のみを提案する。
- `EmergencyCouncil` は **MUST** 緊急時フラグ（fallback/override）のみを操作する。
- `EmergencyCouncil` は **MUST NOT** 恒久的な tokenomics パラメータを直接変更する。

## 5. パラメータ更新手順
### 5.1 更新対象
- `10-tokenomics.md` の各種率・上限
- `12-proposal-evaluation-security.md` の評価/安全パラメータ

### 5.2 フロー
1. `GovCouncil` が proposal 作成
2. `ValidatorSet` が承認
3. timelock 期間待機
4. final checkpoint 境界で適用

### 5.3 timelock
- `TIMELOCK_CHECKPOINTS = 2_880`（初期値）
- timelock 未経過 proposal は **MUST NOT** 適用。

## 6. Solana 障害時 fallback mode
### 6.1 発動条件
次のいずれかで fallback mode 発動候補とする。
1. Solana finality が `T_OUTAGE = 6h` 以上停止
2. Solana 連携 TX が連続 `K_FAIL = 1_000` 失敗

### 6.2 発動手順
- `EmergencyCouncil` は **MUST** `>= 3/5` 署名で発動決議。
- 発動は **MUST** final checkpoint 境界で有効化。

### 6.3 fallback 中の凍結ルール
fallback mode 中は以下を **MUST** 凍結する。
- validator register
- unbond
- slash 実行

凍結対象外:
- 通常 TX 実行
- read-only クエリ

### 6.4 解除条件
次の全条件を満たした場合のみ解除可能。
1. Solana finality 復旧が `T_RECOVER = 12h` 以上継続
2. Solana 連携 TX 成功率が `>= 99%` を 1h 維持
3. `EmergencyCouncil` が `>= 3/5` 署名で解除決議

## 7. emergency local override
### 7.1 発動条件
ノードは次のいずれかで local override を **MAY** 発動。
1. finality 停止 `>= 2h`
2. 同一 `checkpoint_seq` で `state_root` 不一致を検知
3. `validator_set_hash` 不一致を検知
4. `16-bft-liveness-fallback.md` の liveness 停止判定が成立

### 7.2 発動時動作
- ノードは **MUST** 新規 block proposal を停止。
- ノードは **MUST** register/unbond/slash のローカル実行を拒否。
- ノードは **MUST** 既存 final checkpoint までの参照を維持。

### 7.3 解除条件
- `T_STABLE = 4h` の安定運用継続
- `state_root` と `validator_set_hash` 整合性確認

上記成立時、ノードは **MUST** override を解除する。

## 8. 監査ログ
- ガバナンス提案、timelock、fallback 発動/解除、override 発動/解除は **MUST** 監査ログ記録。
- 監査ログは **MUST** `checkpoint_seq` に紐付ける。

## 9. 他仕様参照
- `10-tokenomics.md`
- `12-proposal-evaluation-security.md`
- `02-consensus.md`
- `05-storage-layout.md`
- `16-bft-liveness-fallback.md`
