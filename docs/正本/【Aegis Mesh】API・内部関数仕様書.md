# 【Aegis Mesh】API・内部関数仕様書

更新日：2026-10-16
状態：正本案（クロスレビュー・承認前）

## 0. 目的、根拠、境界
本書は、ChatGPT（設計・管理者）→Aegis Mesh（管制）→Codex親／SubagentsまたはClaude Code（実装）→Aegis Mesh→ChatGPTという初期実証から必須の自動ループを、APIの**論理契約**と内部処理の責務へ具体化する。GitHubは正本、TASK、履歴、Evidence、状態管理のベース、VercelはTypeScript製管制システムと人間用Web画面の実行基盤である。人間によるAI間コピペを前提にしない。

根拠：企画概要書 §3〜20、要件定義書 REQ-WC/REQ-GATE/REQ-RP/REQ-STATE/REQ-CTRL/REQ-CHAT/REQ-TASK/REQ-AUTH/REQ-GH/REQ-CONN、システム仕様書 §4〜23、技術選定理由書 §3〜10。これらが矛盾する場合、本書で独自に解決しない。本書は物理データモデル（データ設計書）、UI（画面仕様書）、認証・実接続経路（外部連携・権限仕様書）を代替しない。

以下の「論理API名」は処理境界の設計名であり、HTTP URL、メソッド、認証方式、Webhook／ポーリング／SDK採用を確定するものではない。これらはU-05〜U-11に連動する。既存の未確定事項を、確定した事実のように書き換えない。

## 1. 共通の論理データ契約

| データ | 必須項目 | 検証 |
|---|---|---|
| Work Contract | project_id,task_id,process_id,context_version,objective,canonical_refs,scope,allowed_actions,forbidden_actions,side_effect_policy,approval_required_if,required_outputs,completion_condition,next_state | 正本SHA・対象範囲・禁止事項・完了条件と整合 |
| Result Packet | project_id,task_id,process_id,context_version,actor,status,claims,changed_files,unchanged_scope,tests,test_results,evidence,diff_summary,side_effects,unresolved,blocked_reason,retry_count,result,next_recommendation | Work Contract／GitHub実体／Evidenceとの整合 |
| 管理イベント | event_id,日時,種別（判断・指示・差戻し・工程判定）,対象工程,参照正本とSHA,判断理由,実施内容,結果,次状態,未決事項 | CHAT_CURRENTに記録・再取得確認 |
| 許可スコープ | scope,allowed_actions,forbidden_actions,side_effect_policy,approval_required_if | actor,repo,branch,path,environment等の必要な範囲を含み越権を否定 |
| Evidence | 対象タスク・工程、証拠の種類、参照先、確認結果 | 結果の主張と実際の証拠の一致 |
| 状態 | 管制：未着手／実行中／確認中／修正必要／BLOCKED／完了／中止。実行主体：待機／実行中／完了／失敗 | 二種類を混同しない。「人間確認待ち」はフラグ |

項目の型・キー・永続化形式はデータ設計書で定義する。未決の項目へ推測で型や値を与えない。

## 2. 論理API一覧
下記のIDと名称は**本書で定義する設計インターフェース**。外部HTTP APIとして公開するか内部境界とするかは、接続方式・認証方式の確定まで保留する。

| ID／論理API | 呼出主体→受け手 | 入力 | 成功結果 | 必須停止・拒否条件 |
|---|---|---|---|---|
| API-01 参照正本取得 | Aegis→GitHub | project_id,canonical_refs | 内容、path、取得版／SHA | 取得不可、許可外、正本不明 |
| API-02 Context照合 | Aegis内部 | Work Contract、最新正本版 | 一致／不一致、根拠 | SHA不一致は実行不可 |
| API-03 管理イベント登録 | ChatGPT→Aegis | 管理イベント | event_id、記録確認結果 | 記録不備、CHAT_CURRENT未反映 |
| API-04 Work Contract受付 | ChatGPT→Aegis | Work Contract、管理event_id | 検証結果、受理可否 | 契約不足、正本不一致、管理イベント未記録 |
| API-05 理解確認 | 実行主体→Aegis | task_id、契約理解結果 | 理解ゲート成立／不成立 | 不明・矛盾・未確認 |
| API-06 実行指示送信 | Aegis→選択した実行系 | 受理済み契約、許可、管理event_id | 送信・受付結果 | 記録未確定、契約不備、理解未成立、権限外 |
| API-07 実行状態受付 | 実行系→Aegis | task_id、actor、管制対象の状態情報、証跡 | 検証・反映結果 | タスク・actor不一致、偽の完了、版不一致 |
| API-08 Result Packet受付 | Codex親／Claude Code→Aegis | Result Packet、Evidence参照 | 照合結果、確認待ち等 | 必須項目欠落、証跡不整合 |
| API-09 管制状態・証跡参照 | ChatGPT／人間向けUI→Aegis | project_id、対象範囲 | **同じ基礎管制データ** | 許可外の参照 |
| API-10 状態遷移申請 | 管理判断→Aegis | task_id、遷移先、管理event_id、根拠 | 遷移成立／拒否 | CHATゲート、契約、状態・Evidence条件未達 |
| API-11 完了判定 | Aegis内部／管理側 | task_id、完了条件、Result Packet、Evidence | 成立／不成立・根拠 | 未試験・未確認・証跡なし・必要承認なし |
| API-12 外部作用許可判定 | 実行系→Aegis | actor、action、scope、target、工程・契約 | 許可／条件不足／人間判断 | 禁止操作、権限外、承認不成立 |
| API-13 人間判断受付 | 人間→Aegis | 対象、判断（許可・却下・再開・中止）、条件 | 記録済み判断 | 操作権・対象不一致 |
| API-14 停止・例外受付 | 実行系／Aegis | task_id、失敗事実、Evidence、原因情報 | 自動継続停止、分析・人間判断待ち | 無根拠の自動Retry／再開禁止 |
| API-15 TASK_CURRENT照合 | Aegis→GitHub | task_id、Result Packet、Evidence | 記録・実体との整合結果 | 未記録や不一致は完了保留 |

「管理イベント登録」と「指示送信」「状態遷移」「完了確定」を別々に外から直接突破できる構造にしない。論理APIを別境界として実装する場合も、後述の管理イベント実行ゲートを全経路で必須にする。

## 3. 内部関数契約
型シグネチャは概念上のもの。実コードの名前・型宣言・HTTP実装は本書のクロスレビュー後、データ設計と整合させる。

| ID／関数 | 入力→出力 | 作用・責任 | 不成立時 |
|---|---|---|---|
| FN-01 loadCanonicalAndVersion | 参照→本文・SHA | GitHubの対象正本を取得 | 取得不能ならSTOP |
| FN-02 validateContextVersion | 契約・最新版→判定 | 開始時と反映前の版を照合 | 不一致ならSTOP |
| FN-03 validateWorkContract | 契約→不足・矛盾一覧 | 必須項目、正本とscopeを検証 | 実行不可 |
| FN-04 verifyUnderstanding | 契約・理解結果→合否 | 実行前理解ゲート | 合格前は指示送信不可 |
| FN-05 appendManagementEvent | イベント→更新結果 | CHAT_CURRENTへ指定内容を追記 | 後続操作不可 |
| FN-06 verifyManagementCommit | event_id、前後SHA→合否 | GitHub再取得で変更とevent_id等の実在を検証 | 後続操作不可 |
| FN-07 requireCommittedManagementEvent | event_id→許可／拒否 | **全ての管理後続操作を解放する共通ゲート** | STOP、送信・遷移・完了不可 |
| FN-08 executeManagementEvent | 管理判断・後続処理→実行結果 | 記録→GitHub検証→FN-07→許可された後続操作の順で実行 | 前半失敗時は後続未実行 |
| FN-09 authorizeSideEffect | actor・action・scope等→許可結果 | 非LLM権限ゲート | 範囲外・禁止を拒否 |
| FN-10 dispatchExecution | 契約・実行系→受付結果 | ChatGPTが選んだ経路に指示し受領記録 | ゲート不成立時は未送信 |
| FN-11 ingestExecutionStatus | 状態報告→検証結果 | 親・Subagent・Claude Code状態を区別 | 無根拠な成功遷移不可 |
| FN-12 validateResultPacket | Packet・契約→不足・不整合 | 必須項目、対象、版、主張を照合 | 完了保留 |
| FN-13 verifyEvidence | 主張・証跡→検証結果 | GitHub差分・試験・反映等の実在を確認 | 不足は完了保留 |
| FN-14 reconcileTaskCurrent | 結果・TASK・GitHub→差異 | 実装側記録と実体を対照 | 不整合は完了保留 |
| FN-15 evaluateCompletion | 完了条件・実行結果・証拠→判定 | 条件成立を機械判定、ChatGPTが意味を判断 | 未達なら自動継続STOP |
| FN-16 evaluateTransition | 現状態・申請・根拠→判定 | 定義された状態遷移のみ許可 | 遷移拒否 |
| FN-17 stopAndClassify | 失敗・原因→停止記録 | 無闇な実装継続を止め、ChatGPT分析へ | 理由不明ならSTOP継続 |
| FN-18 evaluateRetry | 失敗・処理種類・履歴→可否 | 一時的失敗に限り同一処理再試行 | 非一時的失敗はRetry不可 |
| FN-19 evaluateReplan | 原因・履歴→再指示可否 | ChatGPTの原因分析と再計画を扱う | 未解決を勝手に再実行不可 |
| FN-20 routeBlockedDecision | BLOCKED・人間判断→結果 | 人間の再開／中止判断後に必要ゲート再検証 | 判断前の再開禁止 |
| FN-21 buildSharedControlView | 案件・権限→管制データ | UIとChatGPTに共通の状態・証跡を供給 | 取得不能を正常状態と偽装しない |

Retry上限・Failure Signature形式・STOPの状態モデルへの位置付けは、既存U-01〜U-04とシステム仕様書§25に従って未確定のままとする。

## 4. 管理イベント実行の不可逆な順序
1. ChatGPTが判断・指示・差戻し・工程判定の内容と対象を確定する。
2. Aegisがevent_idを関連付けた管理イベントを検証する。
3. `tasks/CHAT_CURRENT.md` の**最新版とSHA**を取得し、既存履歴を保持したまま追記する。
4. GitHubへ書き込み、**再取得して**更新前後SHAと対象event_id・工程・種別・記録内容の存在を検証する。
5. 検証済みのイベントだけを後続処理に渡す。更新失敗時は実行系への送信、差戻し、管制状態遷移、次工程移行、完了確定をいずれも拒否する。
6. 後続処理の許可スコープ・Context Version・理解ゲート・Evidence等、操作別条件をさらに検査し実行する。
7. 結果・失敗を管理履歴と管制データから追跡可能にする。

同一イベントの重複書き込み・二重実行を避ける具体方式はU-09に属する未確定事項。SHA競合が起きたら他者の更新を捨てず停止する。トークンや秘密情報は記録しない。

## 5. 実行結果からChatGPTへ戻る閉ループ
ChatGPTの指示をAegisが正本・契約・履歴・権限で検証→実装系（初期のCodex親／必要工程のClaude Code）を起動→Codex親がSubagentへ分解し結果集約→構造化した状態・Result Packet・EvidenceをAegisが受付・GitHub実体と照合→同じ管制データを人間とChatGPTが参照→ChatGPTが続行・STOP・再指示・人間判断を決める。人間による転記を成功条件にしない。

## 6. 異常時契約・試験観点
- 正本未読／Context Version不一致／契約欠落／理解ゲート不合格：指示未送信。
- CHAT_CURRENTの更新成功がGitHubで確認できない：管理後続操作はすべて不実行。
- 権限外操作・Codexからのデプロイ：拒否。Claude Codeも必要条件未達なら拒否。
- Result Packet欠落、Evidence不一致、テスト未実施：完了確定不可。
- 完了条件未達：自動継続を停止しChatGPTが原因分析。STOPの正式な実装扱いはU-01。
- Retryは一時的エラー以外禁止。再計画が必要ならChatGPTへ。BLOCKED解除は人間判断。
- 複数AIによる並行GitHub更新競合：SHA照合・停止・報告。他者の変更を消さない。
- 計測不能・接続不能を「正常」「完了」に変換しない。

## 7. 未確定事項と後続正本
既存 `docs/計画/【Aegis Mesh】未確定事項確認書.md` U-01〜U-14を参照。特にU-05〜U-11（実接続・認証・状態保存）とU-01〜U-04（STOP・失敗・再試行）が未解決である。公開APIのHTTPメソッド／URL、エラーコード、型・スキーマ、冪等性キー、再送方式、取得頻度、具体の認証・実行トリガーは**本書で確定していない**。実装指示書を作る前に該当論点を確定し、本書とデータ設計書に整合反映する。

本書は作成時点でクロスレビュー・承認前であり、承認済み正本の仕様を置き換えない。
