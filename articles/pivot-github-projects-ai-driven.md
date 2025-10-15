---
title: "AIファーストチームへの変革 - NotionからGitHub Projectsへ移行した理由と実践"
emoji: "🤖"
type: "tech"
topics:
  - "github"
  - "projectmanagement"
  - "ai"
  - "dx"
  - "zennfes2025ai"
published: false
publication_name: "pivotmedia"
---

こんにちは。PIVOTでソフトウェアエンジニアとして、Webフロントエンド、バックエンド、インフラを横断的に担当している[@tawachan](https://x.com/tawachan39)です。

この記事では、PIVOTのプロダクトチームがNotionからGitHub Projectsへタスク管理を移行した経緯と、その背景にある「AIファーストチーム」という考え方について紹介します。

## 開発の最適化から、チーム運営の最適化へ

### 開発ツールのAI化は進んだ - エージェントの時代へ

2024年3月、Devinの発表が日本でも大きな話題となりました。完全自律型のAIソフトウェアエンジニアとして紹介され、「AIがコードを書く」という新しい時代の幕開けを感じさせました。

これまでのGitHub Copilotのような**補完系のツール**から、**エージェントとして自律的にコードを書くツール**への転換点でした。その後、Cursor、Windsurf、Clineなど、エージェント的にコードを生成するツールが次々と登場しました。Ubie社が2024年12月に公開した記事では、DevinにテストコードをCIが通るまで書かせる実践例が紹介されるなど、エージェント活用が現実のものとなっていきました。

そして2025年2月、GitHub Copilot自身も**Agent Mode**（エージェントモード）をプレビューリリース（6月一般提供）。AIが開発者の意図を理解し、自律的にタスクを実行する仕組みが、ついに公式ツールにも組み込まれました。

**AIエージェントがコードに入ってくる**。この流れが、開発現場を大きく変えていきました。

私たちのチームでも、**他社も多分に漏れず**、こうした新しいツールが出てくるたびに**チームとして積極的に試す文化**を作ってきました。個人で興味がある人が率先して情報を拾い、「これは良さそう」というものはチームでも柔軟に試せるようにする。そういった形で継続的に検討してきました。

開発そのものの効率化については、エンジニアなら各自で試行錯誤しているはずです。これらのツールをどう活用するか、どう開発効率を上げるか、という話はまた別の機会に譲ります。

### 次のボトルネックはチーム運営

開発部分がAIで最適化されていくと、次に見えてくるのは**開発以外のボトルネック**です：

- タスク管理の手間
- コミュニケーションコスト
- 意思決定プロセスの遅延
- ドキュメント作成・更新の負担

AIを使えば個人の開発速度は上がります。しかし、**チームとしての動き方がAI時代に最適化されていないと、そこがボトルネックになる**のは当然の流れです。

### AIファーストチームという問い

2025年初、私たちは「AIファーストのチームとは何か」という問いを持ち始めました。単にAIツールを導入するだけでなく、**AI活用を前提としたチーム運営に変えていく必要がある**と考えました。

その一環として決断したのが、**NotionからGitHub Projectsへの移行**です。

## Notionの課題 - AI時代のワークフローに合わない

### MCPはあるが操作しづらい

NotionにはModel Context Protocol (MCP)のサポートがあり、AIからの操作は可能です。しかし、実際に使ってみると以下の課題がありました：

- **更新系の操作が弱い**: データの読み取りは改善されたものの、作成・更新がまだ使いづらい
- **構造化データの扱いづらさ**: データベースの操作がコマンドラインから直感的でない
- **開発ワークフローとの分離**: コードとタスクが別々に管理され、連携に手間がかかる

### 開発ワークフローとの二重管理

Notionで管理していると、以下のような二重管理が発生します：

- GitHub IssueとNotionタスクの同期
- PRとNotionページのリンク管理
- コードレビューとタスクステータスの更新

これらは手動で行う必要があり、**AIで開発効率が上がった分、タスク管理の手間が相対的に大きくなっていました**。

## GitHub Projectsを選んだ理由

### 1. AI親和性 - `gh`コマンドでシュッと操作

GitHub Projectsの最大の強みは、**`gh` CLIでの操作性**です：

```bash
# タスク作成
gh issue create --repo OWNER/REPO --title "タスク" --body "詳細"

# プロジェクトに追加
gh project item-add 1 --owner OWNER --url [ISSUE_URL]

# ステータス変更
gh project item-edit --id [ITEM_ID] --field-id [FIELD] --single-select-option-id [STATUS]
```

Claude CodeやCodex CLIから自然に操作できます。また、Copilot CLIの公式MCPサポートも発表されており、今後さらに統合が進むことが期待できます。

### 2. 低い導入ハードル

GitHub Projectsは、開発チームにとって導入ハードルが非常に低いです：

- **全員がアカウントを持っている**: 開発者なら当然GitHubアカウントがある
- **Issueの経験がある**: 新しい概念を学ぶ必要がない
- **環境設定が最小限**: `gh` CLIの認証を通すだけ（すでにGitHubにアクセスしているなら追加設定ほぼなし）

### 3. シンプルさ - 必要十分な機能

Notionは多機能ですが、**小規模チームには機能が多すぎる**側面もありました。GitHub Projectsは：

- **必要十分な機能**: タスク管理に特化
- **学習コストが低い**: チーム全員がすぐに使える
- **開発ワークフローと統合**: PR、Issue、Projectsがシームレスに連携

## 実装の全体像 - タスク管理リポジトリとAI支援

### タスク管理の統一: `pivot-project-management`

私たちは、すべてのタスクを一つのリポジトリで管理する方針にしました：

- **リポジトリ**: `PIVOT-media/pivot-project-management`
- **すべてのタスク**: iOS、Android、Server、Web、Infra、CMS、QA
- **Task Typeフィールド**: 分野ごとに分類
- **一元管理のメリット**: 情報が散らばらない、横断的な視点を持ちやすい

### Issueテンプレートによるフォーム起票

GitHub Issueテンプレートを用意し、WebUIからもCLIからも起票しやすくしています：

```text
.github/ISSUE_TEMPLATE/
├── general-task.yml          # 一般タスク用
├── bug_report.md             # バグレポート用
├── test-design.md            # テスト設計用
├── test-execution.md         # テスト実施用
└── release-task.yml          # リリース確認用
```

各テンプレートにはデフォルトプロジェクト設定が含まれており、Issue作成時に自動でプロジェクトに追加されます。

**general-task.ymlの例**（簡略版）：

```yaml
name: 一般タスク
description: 通常の開発タスク用
body:
  - type: input
    id: title
    attributes:
      label: タスク概要
    validations:
      required: true
  - type: textarea
    id: description
    attributes:
      label: 詳細
projects: ["PIVOT-media/1"]  # デフォルトでプロジェクト追加
```

これにより、**フォーム入力でもCLIでも柔軟に起票**できます。

### AGENTS.mdによるAI操作最適化

リポジトリに`AGENTS.md`というドキュメントを配置し、**AI（Claude Code/Codex CLI）が効率的に操作できるようにガイド**を整備しています。

**AGENTS.mdに含まれる情報**（要点抜粋）：

#### プロジェクト構成

```markdown
## 現在のプロジェクト構成

### プロダクトチームタスクボード (Project #1)

- **ID**: `PVT_kwDOB01Zus4A5tAH`
- **可視性**: Private（組織内限定）

### ステータス設定

| ステータス | 説明 |
|-----------|------|
| Inbox | タスクを追加したら最初はこのステータス |
| Todo | 直近着手予定 |
| In Progress | 進行中 |
| In Review | レビュー中 |
| Done | 完了 |
| QA | QA中 |
| Next Release | リリース待ち |
```

#### フィールドID情報

```markdown
## 重要なID・識別子

- **プロジェクトID**: `PVT_kwDOB01Zus4A5tAH`
- **Status Field ID**: `PVTSSF_lADOB01Zus4A5tAHzgudNEs`
- **Task Type Field ID**: `PVTSSF_lADOB01Zus4A5tAHzg1ZsTM`

### ステータスオプションID

- **Inbox**: `6ddf110d`
- **Todo**: `f75ad846`
- **In Progress**: `47fc9ee4`
- **Done**: `98236657`

### Task Typeオプション

- **iOS**: `7d251ba0`
- **Android**: `fbbc3648`
- **Web**: `d29e97c9`
- **Infra**: `1d77f93d`
```

#### よく使うコマンドテンプレート

```bash
# Issue作成とプロジェクト追加
gh issue create --repo PIVOT-media/pivot-project-management \
  --title "タスクタイトル" \
  --body "詳細"
gh project item-add 1 --owner PIVOT-media --url [ISSUE_URL]

# ステータス変更
gh project item-edit --id [ITEM_ID] \
  --project-id PVT_kwDOB01Zus4A5tAH \
  --field-id PVTSSF_lADOB01Zus4A5tAHzgudNEs \
  --single-select-option-id f75ad846  # Todo
```

#### Claude Code操作TIPS

```markdown
### Claude Codeでの操作TIPS

#### 並列実行の活用

複数の独立した操作は一つのコマンドで並列実行:

```bash
gh issue create --repo PIVOT-media/pivot-project-management --title "Task1" && \
gh issue create --repo PIVOT-media/pivot-project-management --title "Task2"
```

#### エラーハンドリング

段階的な確認アプローチ:
1. 認証確認 → 2. プロジェクト存在確認 → 3. 操作実行
```

このように、AIが参照しやすい形で情報を整理しています。

### team-members.mdでメンバー情報管理

チームメンバーの情報も構造化してリポジトリに配置しています：

**team-members.mdの例**（簡略版）：

```markdown
# チームメンバー一覧

| GitHubユーザー名 | 本名 | ニックネーム・あだ名 |
|----------------|------|-------------------|
| @kazyam | 山﨑 一人 | kazyam, さきさん |
| @tawachan | 大多和 祐介 | tawachan |
| @indiamela | 楠瀬 大志 | indiamela, インディさん |
| @nk-5 | Keigo Nakagawa | nk5, nk-5, 中川さん |
```

これにより、AIに「インディさんにアサインして」と指示すると、`team-members.md`を参照して適切に`@indiamela`にアサインできます。

## 実際の使い方 - AIを活用したタスク管理

### 基本的なワークフロー

#### 1. gh CLI認証設定

```bash
# GitHub Projectsを操作するためのスコープ追加
gh auth refresh -h github.com -s project

# 認証確認
gh auth status
```

#### 2. タスク作成

```bash
# pivot-project-managementでIssue作成
gh issue create --repo PIVOT-media/pivot-project-management \
  --title "新機能: ユーザープロフィール画面" \
  --body "## 概要
ユーザーのプロフィール情報を表示する画面を実装する

## 詳細
- ..."

# プロジェクトに追加
gh project item-add 1 --owner PIVOT-media --url https://github.com/PIVOT-media/pivot-project-management/issues/XX

# アサイン
gh issue edit XX --repo PIVOT-media/pivot-project-management --add-assignee tawachan
```

#### 3. Task TypeとStatus設定

```bash
# Task TypeをWebに設定
ISSUE_NUMBER=155
gh project item-edit \
  --id $(gh api graphql -f query='{repository(owner:"PIVOT-media",name:"pivot-project-management"){issue(number:'$ISSUE_NUMBER'){projectItems(first:10){nodes{id project{id}}}}}}' --jq '.data.repository.issue.projectItems.nodes[] | select(.project.id == "PVT_kwDOB01Zus4A5tAH") | .id') \
  --project-id "PVT_kwDOB01Zus4A5tAH" \
  --field-id "PVTSSF_lADOB01Zus4A5tAHzg1ZsTM" \
  --single-select-option-id "d29e97c9"
```

### AIを活用したタスク管理

Claude CodeやCodex CLIを使うと、**自然言語でタスク管理**ができます：

#### Slackメッセージからタスク作成

```text
ユーザー: 「このSlackのやり取りをIssueにして」

[Slackメッセージをコピペ]
---
@tawachan: 次のリリースまでにユーザープロフィール画面を実装したい
@kazyam: Androidも対応必要？
@tawachan: まずはWebから。Androidは次スプリントで
---

AI: Issue #156を作成しました。Task TypeをWeb、ステータスをTodo、@tawachanにアサインしました。
```

#### 議論の文字起こしからタスク作成

```text
ユーザー: 「この議事録から実装タスクを抽出してIssueにして」

[議事録テキスト]

AI: 以下の3つのIssueを作成しました：
- Issue #157: ユーザープロフィール画面UI実装 (@tawachan)
- Issue #158: プロフィールAPI実装 (@adachic)
- Issue #159: プロフィール画面のテスト設計 (@KeisukeQA)

それぞれTask Typeとアサインを設定済みです。
```

#### ニックネームでのアサイン

```text
ユーザー: 「このタスクをさきさんにアサインして」

AI: team-members.mdを参照して、@kazyamにアサインしました。
```

このように、**AIがAGENTS.mdとteam-members.mdを参照して適切に操作**してくれます。

## 移行後の変化 - チーム全体の効率化

### 開発者体験の向上

#### コード連携の自然さ

- **PR ↔ Issue自動リンク**: `Closes #123`でPRとIssueが自動連携
- **レビューフローの統合**: コードレビューとタスクステータスがシームレス
- **二重管理からの解放**: NotionとGitHubを行き来する必要がなくなった

#### AI支援の活用

- **タスク管理の民主化**: エンジニア以外もAI経由でタスク操作可能
- **議事録やSlackからの自動起票**: 会議後にすぐタスク化
- **ニックネームでのアサイン指示**: チーム内の呼称で自然に指示できる

### チーム全体の効率化

#### 情報の一元化

- すべてのタスクが一箇所に
- プロジェクト横断的な視点を持ちやすい
- 検索・フィルタリングが容易

#### 透明性の向上

- 誰が何をやっているか一目瞭然
- ステータスの可視化
- 履歴が自動で残る

#### 学習コストの低減

- 開発者なら既存知識で使える
- 新メンバーのオンボーディングが簡単
- シンプルな操作体系

## まとめ - AI時代のチーム運営

### ツール選定の新しい判断軸

AI時代におけるツール選定では、**「AIが操作しやすいか」という視点**が重要になってきています：

- コマンドライン操作性
- 構造化されたAPI
- ドキュメント化のしやすさ

Notionは多機能で素晴らしいツールですが、**小規模開発チームのAIファーストな運用**においては、GitHub Projectsの方がフィットしました。

### シンプルさが新しい価値

過度な機能は、AI時代においてはむしろ足かせになることがあります。**シンプルで明確なデータ構造**の方が、AIにとっても人間にとっても扱いやすいのです。

### 小規模チームだからこそ

小規模チームの強みは**フットワークの軽さ**です：

- 新しいツールへの移行判断が速い
- チーム全員での試行錯誤がしやすい
- フィードバックループが短い

この強みを活かして、AI時代に最適なチーム運営を模索していくことが重要だと感じています。

### 今後の展望

- **Copilot CLIのMCPサポート**: さらなるAI統合の可能性
- **GitHub Projects v2の進化**: 機能追加への期待
- **AI前提のチーム運営**: さらなる改善の余地

開発そのものの効率化だけでなく、**チーム運営全体をAI前提で最適化していく**。これが、AIファーストチームへの変革の本質だと考えています。

この記事が、同じような課題に取り組むチームの参考になれば幸いです。もし質問やフィードバックがあれば、ぜひコメントやXでお気軽に声をかけてください。
