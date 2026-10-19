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
最新の `docs/計画/【Aegis Mesh】未確定事項確認書.md` U-01〜U-19を参照。特にU-05〜U-11（実接続・認証・状態保存）とU-01〜U-04（STOP・失敗・再試行）が未解決である。公開APIのHTTPメソッド／URL、エラーコード、型・スキーマ、冪等性キー、再送方式、取得頻度、具体の認証・実行トリガーは**本書で確定していない**。実装指示書を作る前に該当論点を確定し、本書とデータ設計書に整合反映する。

本書は作成時点でクロスレビュー・承認前であり、承認済み正本の仕様を置き換えない。

## 2026-10-19 承認判断の反映（8正本クロスレビュー）

この節はユーザー承認済みの方針を反映する。以前の記述と矛盾する場合、**現行の実行禁止を緩めず**、ここで明示した将来方針・保存先・判断ルールを優先して整合修正する。実装・自動接続の実証が完了したという意味ではない。

- ChatGPTは管理判断・実装指示・CHAT_CURRENTの内容に責任を持つ。Aegis Meshは承認された管理イベントをGitHubへ反映し、再取得・SHAで検証する。実装側のTASK_CURRENTは、Codex経路ではCodex親がSubagents結果を統合して記録し、Claude Code経路では担当実装者が記録する。
- **仕事票（Work Contract）で依頼し、結果報告票（Result Packet）とEvidenceで結果を返す**既定の方式を維持する。契約提示→担当AIの理解確認→許可条件検証→実行開始の順とし、契約を受け取る前に理解確認を要求しない。Codex親は分担先Subagentsにも各自の担当範囲・禁止事項・完了条件を確認させる。
- 1工程につきChatGPT作成の実装指示書1本を基本とする。Codex親は工程の契約内で安全に作業を分割してSubagentsへ並行割当できる。担当範囲・依存関係・同一ファイルやbranchの更新競合を管理し、安全でなければ順次実行にする。別工程の並行実行には工程別の指示書を用いる。初期実証は1案件、画面は複数案件対応可能な設計を維持する。
- 正本・仕事票・結果報告票・管理状態・承認・Retry/BLOCKED/再計画記録・Evidence等の**永続的な保存先はすべてGitHubリポジトリ**とする。補助DB等を前提にしない。頻繁な更新、競合、性能の実証はGitHubで行い、方針変更は人間判断とする。秘密情報そのものはGitHubに保存しない。
- ChatGPTが選択する実行系はCodex親またはClaude Code。通常選択には都度人間承認不要。新規権限・許可外操作が必要な場合のみ必要な人間判断へ戻す。
- 進捗の機械的な受付や結果受領はAegisが検証できる。一方、**ChatGPTの判断・指示・差戻し・次工程移行・完了確定**はCHAT_CURRENTへの記録・GitHub再取得確認を経る。自動状態更新により管理イベントゲートを迂回してはならない。
- mergeは、対象工程での許可、必須チェック・テスト・レビュー、BLOCKEDでないこと、完了を妨げる未解決事項がないこと、禁止事項に抵触しないこと、必要な承認・承認条件成立を確認した場合のみ許可する。
- **条件付き承認は条件の文書化を必須**とする。対象操作、許可範囲、条件、確認方法とEvidence、判断者を記録し、条件成立を機械的に照合できなければ実行しない。確認済み（結果を見た）と承認済み（操作を許可した）は別記録。
- 失敗時はまず自動継続を停止し、一時的障害で同じ正しい処理と確認でき、二重実行の危険がない場合のみ上限付きRetry。指示・設計・コードの問題はChatGPTが分析し再計画（Replan）。解決不能・権限外・判断必要時はBLOCKEDとし、人間が再開か中止を判断。**判断理由・修正指示・BLOCKED解除などの管理履歴はCHAT_CURRENTに残し**、実行の技術的事実はTASK_CURRENT/Evidenceへ残す。Retry／Replan上限の具体値は実証・工程別仕様で決定する。
- Aegisの画面は人間がAIを巡回するためでなく、進捗・停止理由・判断対象を確認するためのもの。取得できない最新状態を成功・正常と偽らず、人間判断の対象・効果を明示し、同一判断の二重受付を防ぐ。
- AI間の仕事票の受け渡し・結果回収という設計を再審議しない。**物理的な自動送受信経路・起動・認証・復旧**について実証が残る。人間のコピー＆ペーストを恒久的な接続方式にしない。

### Codexデプロイの現在ルールと将来解禁（重要）
**現時点ではCodexからのデプロイ禁止を維持する。** ただし恒久禁止ではなく、ChatGPTの指示品質とAegisのチェック体制を確立・実証した後に、Codexも条件付きでデプロイできる状態を正式な将来目標とする。解禁は正本改訂と承認・実証を経て発効させ、現時点のデプロイ権限を拡張しない。

解禁に最低限必要な5条件：
1. ChatGPTの実装指示書に**デプロイ先、対象範囲、必要なテスト、禁止事項、完了条件**が明記されている。
2. Aegis Meshが**指示内容と許可範囲の整合を実行前**に確認する。
3. Codexの**必須テスト完了とEvidence**が確認できる。
4. **デプロイ直前**に対象環境・変更内容・BLOCKEDの有無・許可条件を再確認する。
5. **デプロイ後**に結果を検証し、失敗時には停止してChatGPTが原因を分析する。

これらのうち条件付き承認に該当する場合、**その条件を文書化・記録し、条件成立を確認してから実行する**。現状のClaude Code条件付きデプロイにも同じ原則を適用する。
