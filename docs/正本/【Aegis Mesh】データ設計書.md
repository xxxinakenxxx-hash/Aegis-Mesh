# 【Aegis Mesh】データ設計書

更新日：2026-10-19
状態：正本案（クロスレビュー・承認前）

## 0. 目的と適用範囲
本書は、ChatGPT（設計・管理者）→Aegis Mesh（管制）→Codex親・Subagents／Claude Code（実行系）→Aegis Mesh→ChatGPTの初期実証から必要な閉ループにおいて、**何を同じデータとして保持し、どの根拠で状態・権限・完了を判定するか**を定義する。

参照正本：企画概要書、要件定義書、システム仕様書、技術選定理由書、API・内部関数仕様書、外部連携・権限仕様書（main、2026-10-19確認版）。状態遷移の真偽はシステム仕様書、論理処理境界はAPI・内部関数仕様書、アクセス範囲は外部連携・権限仕様書を正とし、ここで勝手に変更しない。

**確定しているのは、GitHubが正本・履歴・状態管理のベースであること**。補助状態ストア／DB製品、GitHub内の物理ファイル配置、スキーマ形式、保持期間・更新頻度は未確定。以下の「レコード」は論理データ単位であり、DBテーブル・JSONファイル等の採用を意味しない。

## 1. 全体関係
```text
Project
 ├─ CanonicalReference / ContextVersion
 ├─ Process ─ Task ─ WorkContract ─ PermissionScope
 │             ├─ ActorAssignment ─ Actor（Codex親／Subagent／Claude Code）
 │             ├─ ExecutionStatus / ControlStatus
 │             ├─ ResultPacket ─ Evidence
 │             ├─ Unresolved / RetryRecord / FailureRecord
 │             ├─ Approval
 │             └─ ManagementEvent ─ ChatCommitVerification
 └─ SharedControlView（同じ基礎データを人間とChatGPTへ提供）

GitHub：正本、CHAT_CURRENT、TASK_CURRENT、Evidence、実体・履歴
Aegis：データ間の整合、遷移・権限・完了ゲート
```
上図は**論理関係**。各関係の多重度や永続化手段は実装前に確定する。

## 2. 論理エンティティと必須属性
次の属性は、既存正本の必須項目、識別・照合に必要な項目、履歴に結び付ける項目を整理したもの。物理型、NULL制約、インデックス、生成方式は未確定。

| ID | 論理データ | 項目（名前は既存正本に合わせる） | 関係・用途 |
|---|---|---|---|
| D-01 | Project | project_id、参照repository、現在工程、管制状態 | 全ての管制対象の所属。案件を混同させない |
| D-02 | Process | process_id、project_id、工程目的、完了条件、次工程条件 | 工程を飛ばさないための根拠 |
| D-03 | Task | task_id、project_id、process_id、context_version、担当actor、管制状態、実行主体状態、Work Contract参照、Evidence参照、未解決事項、承認状態 | タスクごとの状態・判定を結び付ける |
| D-04 | Actor | actor、主体区分（人間／ChatGPT／Aegis／Codex親／Subagent／Claude Code）、親の関係（Subagentの場合）、割当タスク | 責務と操作主体を識別 |
| D-05 | CanonicalReference | path、commitまたはblob SHA、repository、確認時点 | 正本が何の版だったか特定 |
| D-06 | ContextVersion | context_version、正本参照集合、タスク開始時版、実行・反映前確認結果 | 正本変更時に自動継続させない |
| D-07 | WorkContract | project_id、task_id、process_id、context_version、objective、canonical_refs、scope、allowed_actions、forbidden_actions、side_effect_policy、approval_required_if、required_outputs、completion_condition、next_state | 仕事・範囲・許可・完了条件を固定 |
| D-08 | PermissionScope | scope、allowed_actions、forbidden_actions、side_effect_policy、approval_required_if、必要に応じactor／repository／branch／path／environment／接続先／データ範囲／期限／コスト上限 | 非LLMの外部作用ゲートで照合 |
| D-09 | ActorAssignment / ExecutionStatus | task_id、actor、担当範囲、実行主体状態、状態確認根拠、最終更新 | Codex親と配下Subagents、別経路Claude Codeを区別 |
| D-10 | ControlStatus | project_id、process_id、task_id、管制状態、人間確認待ちフラグ、状態遷移根拠、最終更新 | 管制側の現在地。実行主体状態と混同しない |
| D-11 | ResultPacket | project_id、task_id、process_id、context_version、actor、status、claims、changed_files、unchanged_scope、tests、test_results、evidence、diff_summary、side_effects、unresolved、blocked_reason、retry_count、result、next_recommendation | 実行系の主張を検証可能にする |
| D-12 | Evidence | task_id、process_id、証跡種別、GitHub参照・commit／blob SHA等、対応する主張・検証結果 | 変更・テスト・外部反映・人間判断の証明 |
| D-13 | Unresolved | 対象工程・タスク、未決事項、完了条件への影響、判断待ち、参照根拠 | 未定義を勝手に補完しない |
| D-14 | RetryRecord | task_id、対象処理、一時的失敗の根拠、retry_count、試行結果、打切根拠 | 単なる未完了をRetry扱いしない |
| D-15 | FailureRecord | task_id、process_id、実行対象、エラー種別、原因分類、失敗箇所、Evidence、Failure Signature参照 | 同一失敗・STOP・Replan判断に使用 |
| D-16 | Approval | 対象タスク・工程、要求する操作・範囲、承認者、判断内容・条件、承認証跡 | 人間判断が必要な操作のみ解放 |
| D-17 | ManagementEvent | event_id、日時、種別、対象工程、参照正本とSHA、判断理由、実施内容、結果、次状態、未決事項 | ChatGPT側の判断・指示・差戻し・工程判定 |
| D-18 | ChatCommitVerification | event_id、CHAT_CURRENTの更新前後SHA、再取得確認結果、対象イベントの記録存在、後続解放可否 | 管理履歴の反映が確認されるまで後続禁止 |
| D-19 | TaskCurrentVerification | 対象タスク、TASK_CURRENT参照、ResultPacket・Evidence・GitHub実体の照合結果 | 実装側履歴の不整合を見逃さない |
| D-20 | SharedControlView | 案件・工程、管制状態、各実行主体、結果、テスト、Evidence、Retry、BLOCKED、Unresolved、人間確認待ち、CHAT/TASK記録確認、最終更新 | 人間向け画面とChatGPTが同じ基礎データを読む |

D-01〜D-20の区分は**本書で導入する設計上の整理**で、全てが独立した物理テーブルになるという確定ではない。

## 3. 状態と整合制約
### 3.1 管制状態
許容される名称は `未着手／実行中／確認中／修正必要／BLOCKED／完了／中止`。
実行主体状態は `待機／実行中／完了／失敗`。「人間確認待ち」はフラグであり管制状態の追加ではない。STOPの正式扱いはU-01で未確定。

### 3.2 Work ContractとContext Version
- Taskは、対象正本のpathとSHAを参照し、開始時・実行／反映前の版を検証できなければならない。
- Work Contractがない、理解できない、正本が不一致なら、タスクの実行を許可しない。
- SubagentはCodex親配下の実行主体、Claude CodeはChatGPT選択の別実行経路として識別し、履歴を混ぜない。

### 3.3 管理イベントとCHAT_CURRENT
管理イベントを受けたら `tasks/CHAT_CURRENT.md` の最新版を取得し、履歴を保持して追記する。GitHubを再取得し、event_id・対象工程・種別・更新前後SHAを照合して、初めて後続操作を解放可能とする。管理イベントの永続化と二重実行防止の具体方式はU-16。

### 3.4 結果とEvidence
Result Packet中の `claims`、`tests`、`test_results`、`changed_files`、`side_effects` は、必要なEvidenceとGitHub実体で照合する。「実装済み」「静的確認済み」「テスト済み」「外部反映済み」「実画面確認済み」「人間確認済み」「完了確定」は区別する。必要な証拠が欠ければ完了確定しない。

### 3.5 停止・Retry・再計画・BLOCKED
完了条件未達なら自動継続をSTOPし、ChatGPTが原因を分析。Retryは一時的な同一処理失敗に限定し、Replanは方針・指示の見直しとして区別する。BLOCKED後の再開・中止は人間判断。Failure Signatureの形式・Retry上限・Replan上限はU-02〜U-04。

### 3.6 権限と外部作用
Work Contract・PermissionScope・Approval・管理イベント記録確認・Context Versionを突き合わせて操作を判定。Codexデプロイは禁止。Claude Codeのデプロイは工程の許可とテスト等の条件成立を要する。許可外のデータ削除・接続変更・権限変更を黙認しない。

## 4. 書き込み責務・参照元
| 情報 | 責務 | 正本・照合先 |
|---|---|---|
| プロジェクト正本・Git変更履歴 | 権限内の担当AI／人間 | GitHub mainの対象正本・Git実体 |
| 管理判断・差戻し・工程判定 | ChatGPTが記録責任、Aegisが記録成功を検証 | `tasks/CHAT_CURRENT.md` |
| 実装・試験・変更結果 | Codex親／Claude Code等、実装担当 | `tasks/TASK_CURRENT.md`、Result Packet、Evidence、GitHub実体 |
| 状態遷移・完了条件判定 | Aegisが機械判定、ChatGPTが業務判断 | 契約・状態・証跡・承認 |
| 人間承認 | 人間が判断、Aegisが権限・履歴を照合 | Approvalと対応証跡 |
| 管制画面とChatGPTの状態読取 | Aegisが共通の基礎データを供給 | SharedControlView |

GitHubが管理のベースであることを維持し、補助ストアを自動的に正本へ格上げしない。

## 5. データのライフサイクルと異常条件
1. 正本取得・版照合→Task／Work Contractの対象固定。
2. 管理イベントのCHAT_CURRENT反映と再確認→操作ゲート。
3. Work Contract／理解／権限ゲート成立→実行系へ受け渡し。
4. 実行主体の状態・Result Packet・Evidence受領→整合検証。
5. Aegisが状態・完了判定、ChatGPTが結果解釈→必要なら新しい管理イベント。
6. 不整合、未承認、証跡欠落、Context Version変更→自動継続禁止。必要ならBLOCKED・人間判断。

イベントの同時更新制御、再送・冪等性、GitHub確認後の部分失敗復旧はU-09／U-16。実装がこれらを勝手に簡易化しない。

## 6. 初期実証で必要なデータ検証
- 正本を読めない／版が異なるTaskは実行不可。
- 契約・理解結果なしに実行系が起動しない。
- 管理イベントのGitHub反映がなければ指示や遷移が成立しない。
- Subagent状態とCodex親状態、Claude Codeの状態を識別できる。
- Result Packetだけで完了確定せずEvidence・TASK・GitHub実体で検証する。
- 許可外操作、Codexデプロイ、未承認外部作用を拒否する。
- 同じ基礎状態を人間とChatGPTへ供給できる。
- STOP／Retry／BLOCKEDの記録と原因判断を混同しない。
- 自動ループを人間の転記なしで実証できる。

## 7. 未確定事項・他正本への依存
既存U-01〜U-16を参照。特にU-10（補助ストア）、U-15（物理通信契約）、U-16（管理イベント復旧）と連動する。**新たに論点として区別するもの**：各論理エンティティの物理保存形態／配置／型・識別子・多重度、更新原子性、保持・削除方針（未確定事項確認書U-17に登録する）。

本書はクロスレビュー・承認前の正本案である。データの論理関係を示したことを、未決のDB採用・外部API方式の確定と取り違えない。
