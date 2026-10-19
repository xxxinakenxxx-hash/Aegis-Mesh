# 【Aegis Mesh】要件定義書

更新日：2026-10-16

## 0. 文書の位置づけ

本書は、正本 `docs/正本/【Aegis Mesh】企画概要書.md` を直接の根拠として、Aegis Meshが満たさなければならない要件を定義する正本文書である。

本書では、企画概要書に記載された目的・思想・役割・主要制約・成功条件を、実装可否を判定できる「要件」と「受入条件」に整理する。

本書だけで詳細実装方式を新たに決めない。状態遷移の詳細、データ項目定義、API・内部関数、画面遷移、外部接続・認証、技術選定理由等は、対応する後続正本で定義する。

企画概要書に明記されていない詳細は推測で補完しない。

参照正本：
- `docs/正本/【Aegis Mesh】企画概要書.md`
- 取得時blob SHA：`f7733a1d954cb961625837722c85bd344b6e09fd`

---

# 1. 目的

Aegis Meshは、複数のAIを安全に働かせるための管制システムとして、次を成立させる。

1. 人間、ChatGPT、Codex親、Subagents、Claude Code等が、同じ正本・同じ状態・同じ制約・同じ完了条件・同じ証跡を基準に動ける。
2. 誤った前提、未確認事項、無許可操作、未達状態が次工程へ正しい前提として伝播しない。
3. AIの自己申告やプロンプト遵守だけに依存せず、外部状態・契約・証跡・ゲート・許可範囲で統制する。
4. 人間をAI間の通信係・常時監視係にしない。
5. 設計者や実装者が変わっても、同じ正本から同じ粒度の工程・実装指示・実装へ展開できる。

---

# 2. 対象範囲

## 2.1 対象

初期の対象は、1案件をAegis Meshで管制し、以下を一貫して管理する範囲とする。

- 正本
- 工程・タスク
- 実行主体
- Work Contract
- Context Version
- 理解ゲート
- Result Packet
- 管制状態
- 実行主体状態
- STOP / Retry / Replan / BLOCKED
- 完了条件
- CHAT_CURRENT
- TASK_CURRENT
- Evidence
- 許可スコープ
- 外部作用ゲート
- 人間確認待ち
- 人間とChatGPTが見る共通状態
- 自動接続
- 管制画面

## 2.2 初期段階で対象外とするもの

企画概要書上、最初から全機能・全案件へ広げることは前提としない。

- 複数案件への全面展開
- すべてのAIサービスの統合
- 人間を完全に外す完全自律化
- 全ての業務をAegis Meshから直接実行する構成

これらは、初期実証後の拡張判断対象とする。

---

# 3. 利用主体と責務要件

## REQ-ROLE-01 人間

Aegis Meshは、人間が以下を担当できること。

- 目的・方針の決定
- 重要仕様判断
- 許可範囲の設定
- AIだけでは決められない事項の判断
- 人間確認が必要な工程の確認
- BLOCKED後の再開または中止判断

### 受入条件
人間がAI間のコピー&ペーストや各AI画面の常時巡回をしなくても、必要な判断だけを行えること。

## REQ-ROLE-02 ChatGPT

ChatGPTは設計・管理者AIとして、以下を行えること。

- 正本確認
- 要件整理
- Work Contract作成
- 実行経路選択
- Result Packet解釈
- STOP後の原因分析
- 再指示・再計画
- 工程判定
- CHAT_CURRENT更新

### 受入条件
ChatGPTが実装者の自己申告だけに依存せず、Aegis Meshの共通状態とEvidenceを読んで判断できること。

## REQ-ROLE-03 Codex親

Codex親は現場監督として、以下を行えること。

- Work Contractに基づく作業分解
- Subagentへの割当
- 並列実行
- 進捗統合
- 結果統合
- 実行結果・証跡の返却

### 受入条件
Codex親が仕様を独自変更せず、与えられた範囲内でSubagentsを管理できること。

## REQ-ROLE-04 Subagents

SubagentsはCodex親配下の実装実行系として、調査・実装・テスト・レビュー等を実行できること。

### 受入条件
Subagentsが与えられた範囲外へ勝手に作業を広げないこと。

## REQ-ROLE-05 Claude Code

Claude Codeは、Codex親とは別の実装実行経路として工程単位で選択できること。

### 受入条件
Claude Codeを利用する工程でも、Aegis Meshが状態・Evidence・完了条件・許可範囲を同じ基準で管制できること。

---

# 4. 正本・Context要件

## REQ-CANON-01 正本参照

各工程・タスクは、参照する正本文書を特定できること。

### 必須情報
- path
- commitまたはblob SHA

### 受入条件
どの正本のどの版を前提にした処理かを後から特定できること。

## REQ-CANON-02 Context Version

各タスクはContext Versionを保持できること。

### 受入条件
タスク開始時の正本と、実行・反映直前の正本が一致しない場合、自動継続しないこと。

## REQ-CANON-03 Drive非参照

Aegis MeshはDrive上の旧資料やミラーを正本・実行時参照元として扱わないこと。

### 受入条件
Aegis Meshの正本参照がGitHub上の `docs/正本/` に限定されていること。

---

# 5. Work Contract要件

## REQ-WC-01 Work Contract必須

AI間で仕事を渡す際、自然言語依頼だけに依存せず、工程・タスクごとにWork Contractを持つこと。

### 必須項目
- project_id
- task_id
- process_id
- context_version
- objective
- canonical_refs
- scope
- allowed_actions
- forbidden_actions
- side_effect_policy
- approval_required_if
- required_outputs
- completion_condition
- next_state

### 追加可能項目
- assigned_actor
- repository
- branch
- target_paths
- environment
- evidence_required
- test_required
- deadline等

## REQ-WC-02 Work Contract外の補完禁止

実装者AIは、Work Contractにない仕様を勝手に補完しないこと。

### 受入条件
契約外の仕様・変更が必要になった場合、自動継続せず停止または判断要求へ移ること。

---

# 6. 理解ゲート要件

## REQ-GATE-01 実行前理解確認

実行者はWork Contract受領後、即実装へ進まず、以下を確認すること。

- 目的
- 変更対象
- 変更禁止対象
- 参照正本
- Context Version
- 必須成果物
- 必須テスト
- 完了条件
- 外部作用の許可範囲
- 次の状態

## REQ-GATE-02 不明・矛盾時停止

理解できない、矛盾する、複数解釈が成立する場合は実行しないこと。

### 受入条件
理解ゲート不成立のまま実装開始できないこと。

---

# 7. Result Packet要件

## REQ-RP-01 Result Packet必須

実装者AI・Codex親は、単なる「完了しました」ではなく、Result Packetを返すこと。

### 必須項目
- project_id
- task_id
- process_id
- context_version
- actor
- status
- claims
- changed_files
- unchanged_scope
- tests
- test_results
- evidence
- diff_summary
- side_effects
- unresolved
- blocked_reason
- retry_count
- result
- next_recommendation

## REQ-RP-02 Evidence照合

Result Packet内の主張はEvidenceと照合可能であること。

### 受入条件
Result Packetだけでは完了確定できず、必要Evidenceと照合されること。

---

# 8. 状態管理要件

## REQ-STATE-01 管制状態

Aegis Meshは少なくとも以下の管制状態を表現できること。

- 未着手
- 実行中
- 確認中
- 修正必要
- BLOCKED
- 完了
- 中止

## REQ-STATE-02 実行主体状態

実行主体について少なくとも以下を表現できること。

- 待機
- 実行中
- 完了
- 失敗

## REQ-STATE-03 補助フラグ

`人間確認待ち` を管制状態とは別の追加フラグとして保持できること。

## REQ-STATE-04 状態遷移ゲート

状態遷移は、AIの文章による宣言だけで成立しないこと。

### 受入条件
必要条件とEvidenceがそろっていない状態で、次状態へ遷移できないこと。

---

# 9. STOP / Retry / Replan / BLOCKED要件

## REQ-CTRL-01 完了条件未達時の自動継続停止

完了条件未達時は、同一処理を漫然と続行せず、いったん自動継続を止めること。

## REQ-CTRL-02 原因分析

STOP後はChatGPTが原因を少なくとも以下に分類できること。

- 指示不足
- プロンプト不備
- 実装ミス
- 環境要因
- 仕様判断必要
- その他

## REQ-CTRL-03 Retry制限

Retryは、同じ正しい処理が一時的要因で失敗した場合に限定すること。

### 受入条件
設計・指示・プロンプト・実装方針が正しいか不明な状態で、同じ処理を繰り返さないこと。

## REQ-CTRL-04 Replan

原因が設計・指示・工程にある場合、Retryではなく再計画・再指示として扱うこと。

## REQ-CTRL-05 Failure Signature

同一失敗の反復を判定できるよう、Failure Signatureを保持できる設計とすること。

### 企画概要書上の構成候補
- 工程
- 実行対象
- エラー種別
- 原因分類
- 失敗箇所

具体形式はシステム仕様書で確定する。

## REQ-CTRL-06 BLOCKED

BLOCKEDは、AIだけで自動継続できない状態として扱うこと。

### 再開経路
```text
BLOCKED
  ↓
人間判断
  ↓
再開 / 中止
```

---

# 10. 完了判定要件

## REQ-DONE-01 状態区別

以下を同一視しないこと。

- 作業実行済み
- 静的確認済み
- テスト済み
- 外部反映済み
- 実画面確認済み
- 人間確認済み
- 完了確定

## REQ-DONE-02 完了責任分担

- Aegis Mesh：定義済み完了条件の成立確認
- ChatGPT：結果の意味解釈と次判断
- 人間：人間確認が完了条件に含まれる場合のみ確認
- 実装者AI：Result PacketとEvidenceの返却

## REQ-DONE-03 自己申告完了禁止

実装者AI自身の完了宣言だけで完了確定しないこと。

---

# 11. CHAT_CURRENT / TASK_CURRENT要件

## REQ-CHAT-01 CHAT_CURRENT

`tasks/CHAT_CURRENT.md` は、設計・管理側の判断、指示、差戻し、工程判定の履歴を保持すること。

## REQ-CHAT-02 管理イベント

以下を管理イベントとして扱うこと。

- 判断
- 指示
- 差戻し
- 工程判定

## REQ-CHAT-03 CHAT_CURRENT必須記録

管理イベント発生時、最低限以下を記録すること。

- 日時
- event_id
- 種別
- 対象工程
- 参照正本とSHA
- 判断理由
- 実施内容
- 結果
- 次の状態
- 未決事項

## REQ-CHAT-04 CHAT_CURRENT更新確認

Aegis Meshは更新後のGitHub実体を再確認し、内容とSHA等を用いて更新成功を検証すること。

## REQ-CHAT-05 CHAT_CURRENTゲート

CHAT_CURRENT更新成功を確認できるまで、以下を実行してはならない。

- 実装者AIへの指示送信
- 差戻し実行
- 管制状態遷移
- 次工程移行
- 完了確定

## REQ-TASK-01 TASK_CURRENT

`tasks/TASK_CURRENT.md` は、実装側の対象工程、変更、テスト結果、外部作用、残件、次状態を保持すること。

## REQ-TASK-02 不整合監視

Aegis Meshは、Result Packet、Evidence、TASK_CURRENT、GitHub実体の不整合を監視できること。

---

# 12. Evidence要件

## REQ-EV-01 Evidence保持

工程に応じて少なくとも以下をEvidenceとして扱えること。

- 正本SHA
- commit
- diff
- PR
- test結果
- build結果
- 反映結果
- 生成物
- 画面確認結果
- Result Packet
- CHAT_CURRENT
- TASK_CURRENT
- 人間承認記録

## REQ-EV-02 Evidence不足時の完了禁止

必要Evidenceがない場合、そのEvidenceを完了条件とする工程を完了させないこと。

---

# 13. 許可スコープ要件

## REQ-AUTH-01 許可スコープ

工程またはタスク開始時に最低限以下を定義できること。

- scope
- allowed_actions
- forbidden_actions
- side_effect_policy
- approval_required_if

必要に応じてactor、repository、branch、path、environment、接続先、対象データ、期限、コスト上限等を追加できること。

## REQ-AUTH-02 事前許可

許可範囲内の操作は、個別再承認を不要として自動実行できること。

## REQ-AUTH-03 範囲外停止

許可範囲外または条件不成立の場合、自動実行せず人間判断へ戻すこと。

## REQ-AUTH-04 非LLMゲート

重要な外部作用は、LLMのプロンプト遵守だけでなく、Aegis Mesh側の実行ゲートで許可条件を検証すること。

## REQ-AUTH-05 Codexデプロイの現時点の禁止と将来解禁条件

Codexはデプロイを実行できないこと。

## REQ-AUTH-06 Claude Code条件付きデプロイ

Claude Codeは、以下をすべて満たす工程のみ条件付きでデプロイできること。

- 工程で許可されている
- 必須テスト合格
- 変更範囲内
- BLOCKEDなし
- 禁止事項抵触なし

## REQ-AUTH-07 その他の外部作用

以下を人間判断対象とすること。

- 新規外部接続
- 外部API追加
- パッケージ追加
- 認証・権限変更
- DB / スキーマ変更

既存データ削除は専用の明示指示なしに実行しないこと。

---

# 14. GitHub要件

## REQ-GH-01 GitHubの役割

GitHubを以下の基盤として利用できること。

- 正本
- 工程表
- 実装指示書
- CHAT_CURRENT
- TASK_CURRENT
- Evidence
- commit
- branch
- PR
- test結果
- 変更履歴
- 状態管理のベース

## REQ-GH-02 Git操作

対象repository、branch、工程、変更範囲が明示されている場合、許可範囲内で以下を自動実行できること。

- status / diff / log等の参照
- 作業branch作成
- 対象ファイル作成・更新
- commit
- 通常push
- PR作成

## REQ-GH-03 merge

mergeは条件付きでのみ実行可能とすること。

最低条件：
- 必須チェック合格
- BLOCKEDなし
- 未解決事項が完了条件を妨げていない
- 禁止事項抵触なし

## REQ-GH-04 禁止Git操作

以下を禁止すること。

- force push
- 履歴改変
- repository削除
- branch削除
- 指定外repository変更
- 指定外branch変更
- 指定外file変更

---

# 15. 共通状態要件

## REQ-SHARED-01 人間とChatGPTの共通状態

人間とChatGPTが、同じ基礎管制データを参照できること。

## REQ-SHARED-02 ChatGPTから確認可能な情報

ChatGPTは少なくとも以下を確認できること。

- 現在工程
- 実行主体
- テスト結果
- Evidence
- Retry
- BLOCKED
- 未解決事項
- 人間確認待ち
- 正本不整合
- CHAT_CURRENT更新状況
- TASK_CURRENT更新状況

## REQ-SHARED-03 自己申告非依存

ChatGPTがCodex親の自己申告だけに依存せず、共通状態とEvidenceから判断できること。

---

# 16. 管制画面要件

## REQ-UI-01 専用Web画面

人間向け管制画面をVercel上の専用Web画面として提供すること。

## REQ-UI-02 最低表示項目

最低限、以下を表示できること。

- 案件
- 現在工程
- 管制状態
- 実行主体
- Codex親状態
- Subagent一覧
- Claude Code利用状況
- 現在作業
- 結果概要
- テスト結果
- Evidence
- Retry
- BLOCKED
- 未解決事項
- 人間確認待ち
- CHAT_CURRENT更新状態
- TASK_CURRENT更新状態
- 最終更新

画面構成・画面遷移の詳細は画面仕様・画面遷移書で定義する。

---

# 17. 自動接続要件

## REQ-CONN-01 人間を中継役にしない

Aegis Meshは、人間をAI間の恒常的なコピー&ペースト係にしないこと。

## REQ-CONN-02 基本自動ループ

少なくとも以下の閉ループを自動接続の目標とする。

```text
ChatGPT
  ↓
Aegis Mesh
  ↓
実装実行系
  ↓
Aegis Mesh
  ↓
ChatGPT
```

具体的なAPI、認証、イベント方式、Codex親・Subagent状態取得方式は外部連携・権限仕様書等で確定する。

---

# 18. 技術要件

## REQ-TECH-01 実装言語

実装言語はTypeScriptとする。

## REQ-TECH-02 実行基盤

実行基盤はVercelとする。

## REQ-TECH-03 状態管理基盤

正本・履歴・状態管理のベースはGitHubとする。

## REQ-TECH-04 人間向けUI

人間向けUIはVercel上の専用Web画面とする。

採用理由・比較・制約は技術選定理由書で定義する。

---

# 19. 非機能要件

## REQ-NF-01 再現性

ChatGPTや設計担当が変わっても、正本群から同じ粒度の開発工程表・実装指示書へ展開できること。

## REQ-NF-02 実装者非依存

実装担当がCodex / Claude Code等に変わっても、同じ制約・完了条件で実装できること。

## REQ-NF-03 監査可能性

何を根拠に、誰が、何を実行し、何を確認し、何が未確認で、なぜその状態になったかを後から確認できること。

## REQ-NF-04 コンテキスト分離

案件・工程・タスク単位で正本、許可、状態、Evidenceを分離し、別案件・別工程の情報が混ざらないこと。

## REQ-NF-05 伝播防止

AIが誤った場合でも、その誤りが確認されないまま次工程の正しい前提として伝播しないこと。

## REQ-NF-06 プロンプト非依存

重要な制約を、LLMの注意書きだけに依存しないこと。

## REQ-NF-07 人間負荷抑制

人間がAI間通信、状態集約、常時監視を手作業で行わなくても運用できること。

---

# 20. 初期実証の受入条件

初期実証は1案件を対象とし、少なくとも以下がすべて確認できること。

| ID | 受入条件 |
|---|---|
| AC-01 | 正本未確認では進めない |
| AC-02 | Context Version不一致では進めない |
| AC-03 | Work Contractなしでは実装開始しない |
| AC-04 | 理解ゲートを通らなければ実行しない |
| AC-05 | 未確定事項を推測で補完しない |
| AC-06 | CHAT_CURRENT未更新で指示送信・状態遷移しない |
| AC-07 | Result PacketとEvidenceなしで完了しない |
| AC-08 | Retryを漫然と繰り返さない |
| AC-09 | 完了条件未達時は自動継続を止めて再検討する |
| AC-10 | BLOCKED時に勝手に再開しない |
| AC-11 | 許可範囲外の外部作用を実行しない |
| AC-12 | 人間とChatGPTが同じ状態を見られる |
| AC-13 | 人間がAI間のコピペ係にならない |
| AC-14 | 実行履歴を後から確認できる |

---

# 21. 後続正本で確定する事項

企画概要書では、以下の詳細は未確定である。

- STOPを正式状態とするか制御動作とするか
- Failure Signatureの正式形式
- 処理種類ごとのRetry上限
- Replan上限・人間判断へ戻す条件
- ChatGPTが状態を読む具体方式
- Codex親が状態を更新する具体方式
- Subagent状態取得方式
- 自動接続の具体プロトコル
- 補助状態ストアの要否
- 最初の実証案件

これらは後続正本で確定し、実装時に推測で補完しない。

---

# 22. 正本から実装へのトレーサビリティ要件

各要件は、後続正本・開発工程表・工程別実装指示書・実装まで追跡できること。

少なくとも各要件について以下を対応付ける。

- 要件ID
- 詳細仕様を定義する正本文書
- 実装する工程
- 実装対象
- テスト
- 必要Evidence
- 完了条件

以下の連鎖を切らない。

```text
企画概要書
  ↓
要件定義書
  ↓
詳細正本
  ↓
開発工程表
  ↓
工程別実装指示書
  ↓
実装
  ↓
テスト / Evidence
```

---

# 23. 最終受入基準

Aegis Meshは、最終的に以下を満たしたとき、企画目的を満たす。

1. 正本を読んでいないAIが進めない。
2. 未確定事項を推測で埋められない。
3. 古いContext Versionで進めない。
4. Work Contract外の作業を進められない。
5. 危険操作をLLM判断だけで突破できない。
6. 判断・指示・差戻し・工程判定がCHAT_CURRENTへ残る。
7. 実装結果がTASK_CURRENTとEvidenceへ残る。
8. 完了条件未達を完了扱いできない。
9. 指示不備時に無意味なRetryを増やさない。
10. 同じ失敗の繰り返しを検知できる。
11. 人間とChatGPTが同じ状態を見る。
12. 人間はAI間の通信係ではなく、必要な判断だけを行う。
13. 設計担当が変わっても、正本群から同じ粒度の開発工程表・実装指示書を再構成できる。
14. 実装担当が変わっても、同じ制約・完了条件で実装できる。
15. AIが間違っても、その間違いが次工程の正しい前提として伝播しない。

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
