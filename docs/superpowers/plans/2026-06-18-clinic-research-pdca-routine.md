# 訪問診療クリニック情報収集・PDCAルーティン Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 瀬戸市尾張旭市在宅医療クリニック向けの情報収集・PDCA提案を毎週月曜9:00 JSTに自動実行するクラウドエージェントのルーティンを構築する。

**Architecture:** リポジトリ内に手順書（`ROUTINE.md`）を置き、`schedule`スキルで作成するcronジョブが毎週そのリポジトリを開いて`ROUTINE.md`の手順に従って実行する。状態はGit履歴（`reports/`配下のファイル）のみで管理し、専用の状態ファイルは持たない。

**Tech Stack:** superpowers `schedule`スキル（cronジョブ作成）、WebSearch/WebFetch（情報収集）、Git/GitHub（永続化）、プッシュ通知。

## Global Constraints

- 実行タイミング：毎週月曜 9:00 JST（spec: 実行方式）
- 対象リポジトリ：このリポジトリ（origin: `https://github.com/ysdkzsh-tech/routine_news.git`）（spec: 実行方式）
- トピックローテーション：`CONTEXT.md`記載順の5トピックを固定順で巡回。`topic_index = (reports/内の*.mdファイル数) mod 5`（spec: 週次実行フロー2）
- レポート保存先：`reports/YYYY-MM-DD.md`（spec: 週次実行フロー6）
- レポート完成後は`origin/main`へ自動push、push成功時にユーザーへプッシュ通知（spec: 週次実行フロー7-8）
- 連携先病院名は対外的に明示しない方針を厳守し、レポート内で実名を断定的に書かない（spec: 週次実行フロー4、CONTEXT.md注意事項）
- 人間によるレビュー・承認フローは設けない（スコープ外）

---

## Task 1: ROUTINE.md（クラウドエージェント向け手順書）の作成

**Files:**
- Create: `ROUTINE.md`

**Interfaces:**
- Produces: cronジョブのプロンプトから参照される手順書。Task 3で作成するcronジョブのプロンプトは「`ROUTINE.md`の手順に従う」という形でこのファイルに依存する。

- [ ] **Step 1: ROUTINE.mdを作成する**

以下の内容で `ROUTINE.md` をリポジトリルートに作成する。

```markdown
# 週次ルーティン実行手順（クラウドエージェント用）

このファイルは、毎週月曜9:00 JSTに起動するクラウドエージェントが従う手順です。`CONTEXT.md` と合わせて必ず読むこと。

## 手順

1. `CONTEXT.md` を読み込み、クリニック背景情報と「収集トピック（週ごとにローテーション）」の5項目を記載順に取得する。
2. `reports/` ディレクトリ内の `*.md` ファイル数を `n` として数える（`.gitkeep` は含めない）。`topic_index = n mod 5`（0始まり）として、CONTEXT.mdの5トピックリストの該当インデックスを今週のトピックとする。
3. `reports/` 内のファイルをファイル名（日付）で降順ソートし、最新の1件があれば読み込む。ファイルが無ければ（初回実行）、ステップ4はスキップする。
4. 直近レポートの `## 今週の計画（Plan）` セクションの内容を抽出する。
5. 今週のトピックに関連する最新情報をWebSearch/WebFetchで収集する。出典は固定リストにせず、信頼性の高い一次情報・専門メディアを優先する（本ファイル末尾の「情報源の傾向」を参照）。連携先病院名は対外的に明示しない方針を厳守し、レポート内で実名を断定的に書かない。
6. 以下の構成で `reports/YYYY-MM-DD.md`（実行日の日付、ハイフン区切り）を作成する：
   - `# YYYY-MM-DD 週次レポート：<今週のトピック名>`
   - `## 前週の振り返り（Check / Act）` ※ステップ3で前回レポートが無かった場合はこのセクション自体を省略する
     - `### Check（評価）`：前回のPlanの各項目が実行可能/妥当だったかを評価する
     - `### Act（改善）`：評価を踏まえた改善点
   - `## 情報収集サマリー`：収集した情報を要約し、各項目に出典リンクを明記する。有意な更新が無かった場合は「大きな動きなし」と明記する。
   - `## 今週の計画（Plan）`：今週収集した情報を踏まえた具体タスクレベルの提案
   - `## 実行ステップ（Do）`：Planを実行に移すための直近の具体的アクション
7. 変更を以下のコマンドでコミットする：
   ```
   git add reports/YYYY-MM-DD.md
   git commit -m "週次レポート: YYYY-MM-DD <トピック名>"
   ```
8. `origin/main` にpushする：`git push origin main`
   - pushが失敗した場合（リモートが進んでいる等）は `git pull --rebase origin main` を1回実行し、再度pushする。それでも失敗する場合は、ローカルコミットは残したまま処理を終了し、完了通知の代わりに失敗内容を通知する。
9. push成功後、ユーザーに「今週のトピック名＋情報収集サマリーの要点1〜2行」を完了通知として送る。

## 情報源の傾向（トピック別）

- 在宅医療・腹膜透析（PD）の制度動向・診療報酬改定：厚生労働省、都道府県（愛知県）、関連審議会資料
- 開業準備手続き：自治体・保健所・医療法人関連の公式情報
- ケアマネジャー・訪問看護向け集患・マーケティング手法：業界メディア、ケアマネジャー・訪問看護向け専門誌
- 人材確保：医療系人材紹介・業界ニュース
- 補助金・助成金情報：厚生労働省・経済産業省・愛知県等の公募情報
```

- [ ] **Step 2: ロジックを手動でトレースして検証する**

現在の `reports/` には `.gitkeep` のみで `*.md` は0件のため `n = 0`。`topic_index = 0 mod 5 = 0`。

`CONTEXT.md` の収集トピックリスト1番目は「在宅医療・腹膜透析（PD）の制度動向・診療報酬改定」のため、初回実行時のトピックはこれになる。これがROUTINE.mdのステップ2のロジックと一致することを確認する（前回レポートが存在しないため、ステップ3・前週振り返りは省略される想定）。

期待結果：n=0,1,2,3,4 がそれぞれCONTEXT.mdの1〜5番目のトピックに1対1で対応し、n=5で再び1番目に戻ることを目視確認する。

- [ ] **Step 3: コミットする**

```bash
git add ROUTINE.md
git commit -m "Add ROUTINE.md: weekly cloud agent execution instructions"
```

---

## Task 2: README.mdの更新

**Files:**
- Modify: `README.md`

**Interfaces:**
- Consumes: Task 1で作成した `ROUTINE.md`（参照リンクとして記載）

- [ ] **Step 1: README.mdに実行手順への参照を追加する**

`README.md` の末尾（既存の箇条書きリストの後）に以下を追加する。

```markdown

## ルーティンの実行

- `schedule`スキルで作成したクラウドエージェントのcronジョブが、毎週月曜9:00 JSTに自動起動する。
- 実行時の詳細手順は [`ROUTINE.md`](ROUTINE.md) に記載されている。
- レポート完成後は自動で `origin/main` にpushされ、完了時にプッシュ通知が送られる。
```

- [ ] **Step 2: 変更内容を確認する**

`README.md` を読み、追記が意図通り反映されていることを目視確認する。

- [ ] **Step 3: コミットする**

```bash
git add README.md
git commit -m "Document weekly routine execution in README"
```

---

## Task 3: 週次cronジョブの作成

**Files:**
- (リポジトリファイルの変更なし。`schedule`スキルを通じて外部のcronジョブ設定を作成する)

**Interfaces:**
- Consumes: `ROUTINE.md`（Task 1）、`CONTEXT.md`（既存）
- Produces: 毎週月曜9:00 JSTに起動するcronジョブ（Task 4で手動トリガーして検証する対象）

> **注意：** cronジョブの作成は外部システムに新たな定期実行を登録する、後から元に戻すのに調整が必要な操作である。実行前にジョブの設定内容（スケジュール・対象リポジトリ・プロンプト）をユーザーに提示し、確認を取ること。

- [ ] **Step 1: cronジョブの設定内容を確定する**

以下の内容で`schedule`スキルを使い、cronジョブを作成する。

- ジョブ名：`clinic-research-pdca-weekly`
- スケジュール：毎週月曜 9:00 JST（cron式 `0 9 * * 1`、タイムゾーン `Asia/Tokyo`）
- 対象リポジトリ：このリポジトリ（`routine_news`）
- エージェントへのプロンプト：

```
このリポジトリ（routine_news）を開き、ROUTINE.md に記載された手順を厳密に実行してください。CONTEXT.md も必ず参照してください。レポート作成・コミット・origin/mainへのpushまで完了したら、今週のトピック名と情報収集サマリーの要点1〜2行をプッシュ通知で知らせてください。push に失敗した場合は ROUTINE.md のエラー処理手順に従い、失敗内容を通知してください。
```

- [ ] **Step 2: ユーザーに設定内容を提示して確認を取る**

上記のジョブ名・スケジュール・対象リポジトリ・プロンプトをそのままユーザーに提示し、作成してよいか確認する。

- [ ] **Step 3: `schedule`スキルでcronジョブを作成する**

確認が取れたら、Skillツールで `schedule` を呼び出し、Step 1の内容でジョブを作成する。

- [ ] **Step 4: 作成されたジョブを一覧で確認する**

`schedule`スキル（または`CronList`）でジョブ一覧を取得し、`clinic-research-pdca-weekly` がスケジュール `0 9 * * 1` (Asia/Tokyo) で登録されていることを確認する。

---

## Task 4: 手動トリガーによる動作検証

**Files:**
- (このタスクではファイルを直接編集しない。Task 1で定義したロジックに従い `reports/` 配下に新規ファイルが生成されることを検証する)

**Interfaces:**
- Consumes: Task 3で作成したcronジョブ

- [ ] **Step 1: cronジョブを手動トリガーで1回実行する**

`schedule`スキル（または`RemoteTrigger`）を使い、Task 3で作成した `clinic-research-pdca-weekly` ジョブを今すぐ1回実行する。

- [ ] **Step 2: 実行結果を検証する**

実行完了後、以下を確認する：

1. `reports/` 配下に本日日付の `YYYY-MM-DD.md` が新規作成されている
2. そのファイルの内容が `# YYYY-MM-DD 週次レポート：在宅医療・腹膜透析（PD）の制度動向・診療報酬改定` から始まり、`## 前週の振り返り（Check / Act）` セクションが**含まれていない**（初回実行のため）
3. `## 情報収集サマリー`・`## 今週の計画（Plan）`・`## 実行ステップ（Do）` の3セクションが存在し、情報収集サマリーには出典リンクが含まれている
4. `git log origin/main` （または `git log -1`）でそのファイルを追加するコミットが `origin/main` に反映されていることを確認する
5. 完了のプッシュ通知が届いていることを確認する

- [ ] **Step 3: 不一致があれば修正する**

Step 2のいずれかが期待と異なる場合、`ROUTINE.md`（Task 1）またはTask 3のプロンプトを修正し、再度Step 1〜2を実施する。
