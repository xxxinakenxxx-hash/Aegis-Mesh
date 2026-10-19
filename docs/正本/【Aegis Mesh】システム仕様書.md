# 【Aegis Mesh】システム仕様書

更新日：2026-10-16

## 0. 文書の位置づけ

本書は、以下3点の正本を基礎としてAegis Meshの管制動作を定義する正本文書である。

- `docs/正本/【Aegis Mesh】企画概要書.md`
  - blob SHA：`f7733a1d954cb961625837722c85bd344b6e09fd`
- `docs/正本/【Aegis Mesh】要件定義書.md`
  - blob SHA：`69a259561a73fe88ecc9cf15439c22e8baf4ddf8`
- `docs/正本/【Aegis Mesh】技術選定理由書.md`
  - blob SHA：`c8865c9d5c917e86cc89f8ea3e60b56096c43536`

本書では、状態、状態遷移、Work Contract、Context Version、理解ゲート、Result Packet、Evidence、STOP / Retry / Replan / BLOCKED、完了判定、CHAT_CURRENT / TASK_CURRENT、許可スコープ、外部作用ゲートを定義する。

データ項目の物理構造、API・内部関数、画面詳細、外部接続の具体方式は、それぞれの後続正本で定義する。3点の正本で未確定とされている事項は、本書で推測して確定しない。

---

# 1. システム目的

Aegis Meshは、人間、ChatGPT、Codex親、Subagents、Claude Code等が、同じ正本、同じ管制状態、同じ許可範囲、同じ完了条件、同じEvidenceを基準に仕事を進めるための管制層である。

Aegis Meshは以下をシステム動作として成立させる。

1. 正本を確認できない状態では処理を開始しない。
2. 古いContext Versionでは処理を継続しない。
3. Work Contractがない状態では実装実行系へ作業を渡さない。
4. 理解ゲートを通過しない状態では実装を開始しない。
5. Work Contract外の作業を許可しない。
6. 必要Evidenceがない状態では完了確定しない。
7. 完了条件未達時に漫然と同一処理を続行しない。
8. BLOCKED後にAIだけで再開しない。
9. 許可範囲外の外部作用を実行しない。
10. CHAT_CURRENT更新確認前に後続の管理操作を解放しない。
11. 人間とChatGPTが同じ基礎管制データを参照できる。
12. AIの自己申告だけを完了根拠にしない。

---

# 2. システム構成

```text
人間
  ↕
ChatGPT
  設計・管理者AI
  ↕
Aegis Mesh
  管制層
  ↕                 ↕
Codex親            Claude Code
  ↓
Subagents

Aegis Mesh
  ↕
GitHub
正本 / CHAT_CURRENT / TASK_CURRENT / Evidence / 履歴 / 状態管理のベース

人間
  ↕
Vercel管制画面
```

実装言語はTypeScript、実行基盤はVercel、正本・履歴・状態管理のベースはGitHub、人間向けUIはVercel上の専用Web画面とする。

---

# 3. 主体と責務

## 3.1 人間

- 目的・方針
- 重要仕様判断
- 許可範囲設定
- AIだけでは決められない事項の判断
- 人間確認が完了条件に含まれる工程の確認
- BLOCKED後の再開または中止判断

人間をAI間の恒常的なコピー&ペースト係、常時監視係にしない。

## 3.2 ChatGPT

- 正本確認
- 要件整理
- Work Contract作成
- 実行経路選択
- Result Packet解釈
- STOP後の原因分析
- Replan / 再指示
- 工程判定
- CHAT_CURRENT更新

ChatGPTもAegis Meshの統制対象であり、正本・許可・Evidence・状態遷移条件を飛ばして後続操作を進めることはできない。

## 3.3 Aegis Mesh

- 正本とContext Version
- Work Contract
- 理解ゲート
- 管制状態
- 実行主体状態
- Evidence
- Result Packet
- Retry
- Replan
- BLOCKED
- 完了条件
- 人間確認待ち
- 許可スコープ
- 外部作用
- CHAT_CURRENT更新ゲート
- TASK_CURRENTとの整合
- GitHub実体との整合

を管制する。

## 3.4 Codex親

- Work Contractに基づく作業分解
- Subagent割当
- 並列実行
- 進捗統合
- 結果統合
- Result Packet・Evidence返却

Codex親はWork Contractにない仕様を独自追加しない。

## 3.5 Subagents

Codex親配下で調査、実装、テスト、レビュー等を実行する。割当範囲を越えて作業を広げない。

## 3.6 Claude Code

Codex親とは別経路で工程単位に選択される実装実行系とする。利用時もAegis Meshの状態、Evidence、完了条件、許可スコープを同じ基準で適用する。

---

# 4. 管制単位

```text
Project
  └─ Process
      └─ Task
          └─ Actor / Execution
```

各Taskは、project_id、process_id、task_id、Context Version、Work Contract、実行主体、管制状態、実行主体状態、Evidence、Result Packet、Retry情報、BLOCKED情報、人間確認待ち、CHAT_CURRENT / TASK_CURRENT関連記録と結び付く。

物理データ構造はデータ設計書で定義する。

---

# 5. Work Contract

## 5.1 必須項目

```text
project_id
task_id
process_id
context_version
objective
canonical_refs
scope
allowed_actions
forbidden_actions
side_effect_policy
approval_required_if
required_outputs
completion_condition
next_state
```

必要に応じて以下を追加する。

- assigned_actor
- repository
- branch
- target_paths
- environment
- evidence_required
- test_required
- deadline
- 接続先
- 対象データ
- コスト上限

## 5.2 運用

- 自然言語依頼だけで実装実行系を開始しない。
- Work Contractにない仕様を実装者AIが勝手に補完しない。
- 契約外作業が必要になった場合は自動継続せず、ChatGPTまたは人間判断へ戻す。

---

# 6. 正本とContext Version

Aegis Mesh固有の正本はGitHub `docs/正本/` を正とし、Driveを正本・実行時参照元にしない。

各Taskは参照正本について最低限以下を保持する。

- path
- commitまたはblob SHA

正本版は少なくともTask開始時と実行または反映直前に確認する。

不一致の場合は以下とする。

- 実装開始前：開始しない
- 実装中：後続操作を止める
- 反映直前：反映しない
- ChatGPTへ再確認を要求する

Context Version不一致をAIの判断だけで無視して継続してはならない。

---

# 7. 理解ゲート

実行主体はWork Contract受領後、以下を確認する。

- 目的
- 変更対象
- 変更禁止対象
- 参照正本
- Context Version
- 必須成果物
- 必須テスト
- 完了条件
- 外部作用の許可範囲
- 次状態

以下のいずれかに該当する場合は実行しない。

- 必須項目不足
- 指示の矛盾
- 複数解釈が成立
- 正本を確認できない
- Context Versionを特定できない
- 許可範囲を判定できない
- 完了条件を判定できない

---

# 8. 状態管理

## 8.1 管制状態

```text
未着手
実行中
確認中
修正必要
BLOCKED
完了
中止
```

## 8.2 実行主体状態

```text
待機
実行中
完了
失敗
```

`人間確認待ち` は管制状態とは別のフラグとする。

状態遷移はAIの文章による宣言だけでは成立せず、定義済み条件とEvidenceがそろった場合のみ成立する。

---

# 9. 状態遷移

## 9.1 未着手 → 実行中

以下をすべて満たすこと。

- Work Contractあり
- 正本確認済み
- Context Version有効
- 理解ゲート成立
- 許可スコープ確認済み
- 必要なCHAT_CURRENT更新確認済み
- BLOCKEDではない

## 9.2 実行中 → 確認中

実装実行系から必要なResult PacketとEvidenceが返却された場合。

## 9.3 確認中 → 完了

以下をすべて満たすこと。

- completion_condition成立
- 必須テスト成立
- 必須Evidence存在
- Result PacketとEvidence整合
- TASK_CURRENTとGitHub実体整合
- Context Version整合
- 必要な人間確認完了
- BLOCKEDなし
- CHAT_CURRENT更新ゲート成立

## 9.4 確認中 → 修正必要

完了条件未達であり、かつAI側で原因分析して再指示可能な場合。

## 9.5 修正必要 → 実行中

ChatGPTが原因分析後、必要なWork Contractまたは指示を再定義し、CHAT_CURRENT更新、Context Version確認、理解ゲート、許可確認を再度満たした場合。

## 9.6 BLOCKED

以下の場合に移す。

- 仕様判断が必要
- 方針が一意でない
- 許可範囲外の操作が必要
- 原因特定不能
- 同一失敗の反復
- 必要承認未取得

BLOCKED後はAIだけで自動再開しない。

```text
BLOCKED
  ↓
人間判断
  ↓
再開 / 中止
```

再開時の具体的な遷移先は3点正本では未確定のため本書では固定しない。

---

# 10. STOP / Retry / Replan

## 10.1 STOP

完了条件未達、重大不整合、Context Version不一致等を検知した場合、自動継続を止める。

```text
未達・不整合
  ↓
自動継続停止
  ↓
ChatGPT原因分析
  ↓
Retry / Replan / BLOCKED
```

STOPを正式な管制状態とするか、制御動作とするかは未確定。

## 10.2 Retry

同じ正しい処理が一時的要因で失敗した場合に限定する。

以下ではRetryしない。

- 設計が正しいか不明
- 指示が正しいか不明
- プロンプトが正しいか不明
- 実装方針が正しいか不明
- 仕様判断が必要
- 許可範囲外

処理種類ごとのRetry上限、間隔、打切条件は未確定。

## 10.3 Replan

原因が設計、指示、工程、プロンプト、実装方針にある場合はRetryではなくReplanとする。

ChatGPTが原因分析し、Work Contractまたは指示を再定義する。

Replan上限、人間判断へ戻す具体条件は未確定。

---

# 11. Failure Signature

同一失敗を単なるエラー文一致だけで判定しない。

3点正本で確認できる構成候補は以下。

- 工程
- 実行対象
- エラー種別
- 原因分類
- 失敗箇所

正式形式、正規化方法、比較方法、同一失敗判定条件は未確定。

---

# 12. Result Packet

必須項目：

```text
project_id
task_id
process_id
context_version
actor
status
claims
changed_files
unchanged_scope
tests
test_results
evidence
diff_summary
side_effects
unresolved
blocked_reason
retry_count
result
next_recommendation
```

Aegis Meshは少なくとも以下を照合する。

- Context Version
- changed_files
- tests / test_results
- Evidence
- side_effects
- unresolved
- blocked_reason
- retry_count
- TASK_CURRENT
- GitHub実体

Result Packet単独では完了を確定しない。

---

# 13. Evidence

工程に応じて以下をEvidenceとして扱える。

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

必須Evidenceが欠落している工程は完了させない。

AIの主張とEvidenceが一致しない場合、AIの主張を優先しない。

---

# 14. 完了判定

以下を同一視しない。

```text
作業実行済み
静的確認済み
テスト済み
外部反映済み
実画面確認済み
人間確認済み
完了確定
```

責任分担：

- Aegis Mesh：completion_condition成立確認
- ChatGPT：結果解釈・次判断
- 人間：必要な場合のみ人間確認
- 実装者AI：Result PacketとEvidence返却

実装者AIの完了宣言だけでは完了確定しない。

---

# 15. CHAT_CURRENTゲート

`tasks/CHAT_CURRENT.md` は設計・管理側履歴であり、仕様正本ではない。

管理イベント：

- 判断
- 指示
- 差戻し
- 工程判定

最低記録項目：

- 日時
- event_id
- 種別
- 対象工程
- 参照正本とSHA
- 判断理由
- 実施内容
- 結果
- 次状態
- 未決事項

Aegis Meshは更新後のGitHub実体を再確認し、内容とSHA等で更新成功を検証する。

更新成功を確認できるまで以下を禁止する。

- 実装者AIへの指示送信
- 差戻し実行
- 管制状態遷移
- 次工程移行
- 完了確定

---

# 16. TASK_CURRENT

`tasks/TASK_CURRENT.md` は実装側履歴であり、仕様正本ではない。

工程実行後、少なくとも以下を記録する。

- 対象工程
- 変更ファイル
- テスト結果
- 外部作用
- 残件
- 次状態

Aegis MeshはResult Packet、Evidence、TASK_CURRENT、GitHub実体の不整合を監視し、不整合状態では完了確定しない。

---

# 17. 許可スコープ

最低項目：

```text
scope
allowed_actions
forbidden_actions
side_effect_policy
approval_required_if
```

必要に応じてactor、repository、branch、path、environment、接続先、対象データ、期限、コスト上限等を追加する。

- 事前許可済み：範囲内で自動実行可能
- 条件付き許可：条件成立時のみ自動実行可能
- 範囲外 / 禁止：実行せず人間判断へ戻す

重要な外部作用はLLMの注意書きだけに依存させず、Aegis Mesh側の実行ゲートで検証する。

---

# 18. Git操作・外部作用

対象repository、branch、工程、変更範囲が明示され、許可スコープ内の場合、以下を自動実行可能とする。

- status / diff / log等の参照
- 作業branch作成
- 対象ファイル作成・更新
- commit
- 通常push
- PR作成

mergeは以下をすべて満たす場合のみ条件付きとする。

- 対象工程で許可
- 必須チェック合格
- BLOCKEDなし
- 未解決事項が完了条件を妨げていない
- 禁止事項抵触なし

履歴破壊、repository削除、branch削除、指定外repository / branch / file変更は禁止する。

Codexはデプロイ禁止。Claude Codeは対象工程で許可、必須テスト合格、変更範囲内、BLOCKEDなし、禁止事項抵触なしの場合のみ条件付きデプロイとする。

新規外部接続、外部API追加、パッケージ追加、認証・権限変更、DB / スキーマ変更は人間判断対象。既存データ削除は専用の明示指示なしに実行しない。

---

# 19. 人間とChatGPTの共通状態

人間とChatGPTは同じ基礎管制データを参照できること。

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

具体的な読取方式は未確定。

---

# 20. 自動接続

目標ループ：

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

ChatGPT接続、Codex親更新、Subagent状態取得、Claude Code接続、具体プロトコル等は未確定であり、後続正本で定義する。

---

# 21. 管制画面へ供給する情報

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

画面構成・操作・遷移は画面仕様・画面遷移書で定義する。

---

# 22. GitHubと状態保存

GitHubは正本・履歴・状態管理のベースとして使用するが、Aegis Mesh本体ではない。

GitHubだけでWork Contract、理解ゲート、状態遷移ゲート、Retry / Replan / BLOCKED、完了判定、外部作用ゲート、AI間自動接続、管制UIが成立するとはみなさない。

補助状態ストアの要否・製品は未確定。

---

# 23. 初期実証の受入条件

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

# 24. 後続正本への引き渡し

## データ設計書

Project、Process、Task、Actor、Work Contract、Context Version、Evidence、Result Packet、Unresolved、Approval、Retry、BLOCKED、Event等のデータ構造。

## API・内部関数仕様書

正本取得・版検証、Work Contract作成/検証、理解ゲート、状態遷移判定、Result Packet検証、Evidence検証、完了判定、CHAT_CURRENT更新/検証、TASK_CURRENT整合、許可スコープ、外部作用ゲート、Retry / Replan / BLOCKED処理。

## 画面仕様・画面遷移書

管制画面の表示項目、操作、遷移、人間判断導線。

## 外部連携・権限仕様書

GitHub、Vercel、ChatGPT、Codex、Subagents、Claude Codeとの接続、権限、自動接続方式。

---

# 25. 未確定のまま残す事項

3点正本上で未確定であるため、本書では以下を確定しない。

- STOPを正式状態とするか制御動作とするか
- Failure Signature正式形式
- 処理種類ごとのRetry上限
- Replan上限・人間判断へ戻す具体条件
- BLOCKED解除後の具体遷移先
- ChatGPT状態読取方式
- Codex親状態更新方式
- Subagent状態取得方式
- Claude Code接続方式
- 自動接続プロトコル
- 補助状態ストアの要否・製品
- 最初の実証案件

---

# 26. トレーサビリティ

| 本書仕様 | 要件 |
|---|---|
| Work Contract | REQ-WC-01 / 02 |
| Context Version | REQ-CANON-01 / 02 / 03 |
| 理解ゲート | REQ-GATE-01 / 02 |
| Result Packet | REQ-RP-01 / 02 |
| 状態管理 | REQ-STATE-01〜04 |
| STOP / Retry / Replan / BLOCKED | REQ-CTRL-01〜06 |
| 完了判定 | REQ-DONE-01〜03 |
| CHAT_CURRENT | REQ-CHAT-01〜05 |
| TASK_CURRENT | REQ-TASK-01 / 02 |
| Evidence | REQ-EV-01 / 02 |
| 許可スコープ | REQ-AUTH-01〜07 |
| Git操作 | REQ-GH-01〜04 |
| 共通状態 | REQ-SHARED-01〜03 |
| 管制画面 | REQ-UI-01 / 02 |
| 自動接続 | REQ-CONN-01 / 02 |
| 技術基盤 | REQ-TECH-01〜04 |
| 非機能 | REQ-NF-01〜07 |

---

# 27. 結論

Aegis Meshの基本管制ループは以下とする。

```text
正本
  ↓
Context Version
  ↓
Work Contract
  ↓
理解ゲート
  ↓
許可スコープ確認
  ↓
実行
  ↓
Result Packet + Evidence
  ↓
整合確認
  ↓
完了条件判定
  ↓
状態遷移
```

各段階の条件を満たさなければ自動継続しない。

AIが「理解した」「完了した」と宣言すること自体は、Aegis Meshの管制上の成立条件にはしない。

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
