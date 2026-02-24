# 07 Cross-chain Trust Model 仕様

## 1. 目的と適用範囲
本仕様は Misaka と Solana 間の trust model を定義する。
Phase 1 では semi-trusted モデルを採用し、段階的に trustless へ移行する。

## 2. 規範キーワード
- **MUST**: 準拠実装に必須
- **SHOULD**: 特段の理由がない限り必須
- **MAY**: 任意

## 3. Phase 定義
- **Phase 1 (semi-trusted)**: Misaka finality + External Multisig（M-of-N）で外部実行。
- **Phase 2 (trust-minimized)**: Light client もしくは集約署名検証を導入。
- **Phase 3 (target trustless)**: 外部チェーン上で Misaka finality を暗号学的に単独検証可能。

## 4. Phase 1 信頼境界
### 4.1 信頼前提
Phase 1 では以下を信頼境界とする。
1. Misaka の BFT finality（`02-consensus.md`）
2. Slash Committee の閾値署名（`06-anchor-collateral-phase1.md`）

### 4.2 非目標
- Phase 1 は **MUST NOT** 完全 trustless slashing を主張しない。
- committee 署名なしの自動 slash は **MUST NOT** 実装する。

## 5. 判定主体（曖昧さ排除）
- `slash_proof` の一次整合性（checkpoint/state_root/validator_set_hash）は **MUST** Anchor 側検証ロジックが判定。
- slash 実行承認は **MUST** Slash Committee（M-of-N）が判定。
- Solana Program 実行可否は **MUST** Program 側の on-chain 検証結果で最終決定。

## 6. 署名ルール
### 6.1 最低署名数
- 最低署名数は **MUST** `committee_m` とし、`committee_m >= ceil(2N/3)` を満たす。

### 6.2 署名対象
- 署名対象は **MUST** `06` で定義された `signing_message` と一致。
- 別メッセージ署名は **MUST** 無効。

### 6.3 失敗時
- 署名不足/無効署名は **MUST** slash 不成立。
- proof 検証失敗は **MUST** 実行拒否。

## 7. Solana 側 proof 受け渡し
Solana へ渡す payload は **MUST** 以下を含む。
- `slash_proof_v1` 本体（MCS-1 bytes）
- committee 署名列
- optional: compact evidence bytes

Program は **MUST** 以下を検証する。
1. `proof_format_version == 1`
2. committee 署名が閾値以上
3. nonce 再利用なし
4. expiry 未超過

## 8. Phase 2 への移行条件
Phase 1 から Phase 2 への移行は、以下全条件を **MUST** 満たす。
1. Light client 仕様または集約署名仕様が固定済み
2. 少なくとも 2 実装で相互検証テストに合格
3. 悪性 proof テスト（偽state_root/偽validator_set_hash）で拒否を確認
4. ガバナンス承認 + timelock 経過

## 9. trustless claim 許可条件
「trustless」を名乗るのは以下成立時のみ **MUST** 許可。
- 外部チェーン上で Misaka finality を committee 非依存で検証可能
- 検証に必要な証跡が公開仕様で再現可能
- 監査済み実装が mainnet に反映済み

上記未達時は **MUST** `semi-trusted` 表記を維持する。

## 10. 他仕様参照
- `02-consensus.md`
- `06-anchor-collateral-phase1.md`
- `11-governance-and-emergency-mode.md`
