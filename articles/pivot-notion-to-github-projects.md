---
title: "NotionからGitHub Projectsへ移行した理由と実践"
emoji: "📋"
type: "tech"
topics:
  - "github"
  - "notion"
  - "projectmanagement"
  - "dx"
  - "ai"
published: false
publication_name: "pivotmedia"
---

こんにちは。PIVOTでソフトウェアエンジニアとして、Webフロントエンド、バックエンド、インフラを横断的に担当している[@tawachan](https://x.com/tawachan39)です。

この記事では、PIVOTのプロダクトチームがNotionからGitHub Projectsへタスク管理ツールを移行した経緯と、具体的な実装方法について紹介します。

## 背景 - AIファーストチームへの変革

詳細な背景については、前回の記事「[AI時代のチーム運営 - スプリントベースからカンバンへの移行](リンクを挿入)」で紹介していますが、簡単に振り返ります。

### AIエージェント時代の到来

Ubie社の[Devinの活用事例](https://zenn.dev/ubie_dev/articles/devin-for-test)をきっかけに、AIエージェントが実際に開発現場で使えることが明確になりました。

補完系のツールから、エージェントとして自律的にコードを書くツールへ。この流れが、開発だけでなくチーム運営も変えていく必要性を示していました。

### チーム運営の最適化

開発がAIで最適化されると、次のボトルネックは**チーム運営**です。

その一環として、私たちは：
1. スプリントベースからカンバン方式へ移行
2. タスク管理ツールをNotionからGitHub Projectsへ移行

この記事では、**2番目のツール移行**について詳しく解説します。

## Notionの課題 - フットワークが重くなる

### Notionは優れたツールだが

Notionは非常に優れたツールです：

- 柔軟なドキュメント作成
- データベース機能
- チーム全体での情報共有

私たちもNotionでタスク管理を行っていました。

### しかし、AIファーストの観点で課題が

#### 1. MCPはあるが更新系が弱い

NotionにはModel Context Protocol (MCP)のサポートがあり、AIからの操作は可能です。しかし：

- **データの読み取り**は改善されてきた
- **作成・更新系の操作**はまだ使いづらい
- コマンドラインからの操作が直感的でない

AIエージェントに「このタスクをNotionに追加して」と指示しても、スムーズに動かないことが多々ありました。

#### 2. 開発ワークフローとの分離

Notionで管理していると、以下のような二重管理が発生します：

- **GitHub IssueとNotionタスクの同期**
- **PRとNotionページのリンク管理**
- **コードレビューとタスクステータスの更新**

これらは手動で行う必要があり、フットワークが重くなる要因でした。

#### 3. 開発者にとってのコンテキストスイッチ

開発者の日常：

1. VSCodeでコードを書く
2. GitHub でPRを作る
3. Notionでタスクを更新する ← **ここでコンテキストスイッチ**

NotionのUI を開いて、該当するタスクを探して、ステータスを更新して...という手順が、小さいながらもストレスでした。

#### 4. AI操作のしづらさ

AIエージェント（Claude Code/Codex CLIなど）から操作しようとすると：

- NotionのAPI認証が必要
- データベースIDやページIDを指定する必要がある
- 構造化されたデータの扱いが煩雑

`gh` コマンドのようなシンプルな操作性とは程遠い状況でした。

## GitHub Projectsを選んだ理由

### 1. AI親和性 - `gh`コマンドでシュッと操作

GitHub Projectsの最大の強みは、**`gh` CLIでの操作性**です：

```bash
# Issue作成
gh issue create --title "タスク" --body "詳細"

# プロジェクトに追加
gh project item-add 1 --owner OWNER --url [ISSUE_URL]

# ステータス変更
gh project item-edit --id [ITEM_ID] --field-id [FIELD] --single-select-option-id [STATUS]
```

Claude CodeやCodex CLIから自然に操作できます。

GitHub Copilot自身も**Agent Mode**をリリースし、公式MCPサポートも発表されています。今後さらに統合が進むことが期待できます。

### 2. 開発ワークフローとの統合

GitHub Projectsなら：

- **PR ↔ Issue 自動リンク**: `Closes #123`で自動連携
- **コードレビューとタスクが同じ場所**
- **二重管理からの解放**

開発者はGitHubから離れる必要がなくなります。

### 3. 低い導入ハードル

- **全員がアカウントを持っている**: 開発者なら当然GitHubアカウントがある
- **Issueの経験がある**: 新しい概念を学ぶ必要がない
- **環境設定が最小限**: `gh` CLIの認証を通すだけ

### 4. シンプルさ

Notionは多機能ですが、**小規模チームには機能が多すぎる**側面もありました。

GitHub Projectsは：

- **必要十分な機能**: タスク管理に特化
- **学習コストが低い**: チーム全員がすぐに使える
- **シンプルな操作体系**

複雑さを避け、必要なことだけを効率的にできます。

## 実装の全体像

### タスク管理の統一: `pivot-project-management`

すべてのタスクを一つのリポジトリで管理する方針にしました：

- **リポジトリ**: `PIVOT-media/pivot-project-management`
- **すべてのタスク**: iOS、Android、Server、Web、Infra、CMS、QA
- **Task Typeフィールド**: 分野ごとに分類

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

```markdown
## 現在のプロジェクト構成

### プロダクトチームタスクボード (Project #1)

- **ID**: `PVT_kwDOB01Zus4A5tAH`
- **ステータス**: Inbox/Todo/In Progress/In Review/Done/QA/Next Release

### 重要なID・識別子

- **プロジェクトID**: `PVT_kwDOB01Zus4A5tAH`
- **Status Field ID**: `PVTSSF_lADOB01Zus4A5tAHzgudNEs`
- **Task Type Field ID**: `PVTSSF_lADOB01Zus4A5tAHzg1ZsTM`

### よく使うコマンドテンプレート

```bash
# Issue作成とプロジェクト追加
gh issue create --repo PIVOT-media/pivot-project-management \
  --title "タスクタイトル" \
  --body "詳細"

# ステータス変更
gh project item-edit --id [ITEM_ID] \
  --project-id PVT_kwDOB01Zus4A5tAH \
  --field-id PVTSSF_lADOB01Zus4A5tAHzgudNEs \
  --single-select-option-id f75ad846  # Todo
```

AIが参照しやすい形で情報を整理しています。

### team-members.mdでメンバー情報管理

チームメンバーの情報も構造化してリポジトリに配置：

```markdown
# チームメンバー一覧

| GitHubユーザー名 | 本名 | ニックネーム・あだ名 |
|----------------|------|-------------------|
| @kazyam | 山﨑 一人 | kazyam, さきさん |
| @tawachan | 大多和 祐介 | tawachan |
| @indiamela | 楠瀬 大志 | indiamela, インディさん |
```

これにより、AIに「インディさんにアサインして」と指示すると、`team-members.md`を参照して適切に`@indiamela`にアサインできます。

## 実際の使い方 - AIを活用したタスク管理

### 基本的なワークフロー

#### gh CLI認証設定

```bash
# GitHub Projectsを操作するためのスコープ追加
gh auth refresh -h github.com -s project

# 認証確認
gh auth status
```

#### タスク作成

```bash
# pivot-project-managementでIssue作成
gh issue create --repo PIVOT-media/pivot-project-management \
  --title "新機能: ユーザープロフィール画面" \
  --body "## 概要
ユーザーのプロフィール情報を表示する画面を実装する"

# プロジェクトに追加
gh project item-add 1 --owner PIVOT-media --url [ISSUE_URL]

# アサイン
gh issue edit XX --repo PIVOT-media/pivot-project-management --add-assignee tawachan
```

### AIを活用したタスク管理

Claude CodeやCodex CLIを使うと、**自然言語でタスク管理**ができます：

#### Slackメッセージからタスク作成

```text
ユーザー: 「このSlackのやり取りをIssueにして」

[Slackメッセージをコピペ]

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
```

#### ニックネームでのアサイン

```text
ユーザー: 「このタスクをさきさんにアサインして」

AI: team-members.mdを参照して、@kazyamにアサインしました。
```

このように、**AIがAGENTS.mdとteam-members.mdを参照して適切に操作**してくれます。

## 移行後の変化

### 開発者体験の劇的な向上

#### コード連携の自然さ

- **PR ↔ Issue自動リンク**: `Closes #123`でPRとIssueが自動連携
- **レビューフローの統合**: コードレビューとタスクステータスがシームレス
- **二重管理からの解放**: NotionとGitHubを行き来する必要がなくなった

#### コンテキストスイッチの削減

開発者の新しい日常：

1. VSCodeでコードを書く
2. `gh issue create` でタスク作成
3. GitHubでPRを作る
4. 全てがGitHub上で完結

Notionを開く必要がなくなり、開発フローが途切れません。

### AI支援の活用

- **タスク管理の民主化**: エンジニア以外もAI経由でタスク操作可能
- **議事録やSlackからの自動起票**: 会議後にすぐタスク化
- **ニックネームでのアサイン指示**: チーム内の呼称で自然に指示できる

### チーム全体の効率化

#### 情報の一元化

- すべてのタスクがGitHub上に
- コードとタスクが同じ場所
- 検索・フィルタリングが容易

#### 透明性の向上

- 誰が何をやっているか一目瞭然
- ステータスの可視化
- 履歴が自動で残る

#### 学習コストの低減

- 開発者なら既存知識で使える
- 新メンバーのオンボーディングが簡単
- シンプルな操作体系

## Notionからの移行手順

### 1. 既存タスクの棚卸し

Notionの全タスクをレビューし：

- **継続するタスク**: GitHub Issueに移行
- **完了済みタスク**: そのまま残す（履歴として）
- **不要なタスク**: クローズ

### 2. GitHub Projects設定

- プロジェクトを作成
- フィールド設定（Status、Task Type等）
- ビューの設定（カンバンボード等）

### 3. Issueテンプレート作成

よく使うタスクタイプごとにテンプレートを用意。

### 4. ドキュメント整備

- `AGENTS.md`: AI操作ガイド
- `team-members.md`: メンバー情報
- `README.md`: リポジトリの説明

### 5. チーム内周知

- 移行の目的と背景を共有
- 使い方のガイド
- 質問対応

### 6. 段階的移行

- 新規タスクから GitHub Projects へ
- Notionは並行運用期間を設けて徐々に移行

## まとめ - ツール選定の新しい判断軸

### AIが操作しやすいか

AI時代におけるツール選定では、**「AIが操作しやすいか」という視点**が重要になってきています：

- コマンドライン操作性
- 構造化されたAPI
- ドキュメント化のしやすさ

Notionは多機能で素晴らしいツールですが、**小規模開発チームのAIファーストな運用**においては、GitHub Projectsの方がフィットしました。

### シンプルさが新しい価値

過度な機能は、AI時代においてはむしろ足かせになることがあります。**シンプルで明確なデータ構造**の方が、AIにとっても人間にとっても扱いやすいのです。

### チーム状況に応じた選択

重要なのは：

- ツールの機能の多さではない
- **チームの状況に合っているか**
- **AIと統合しやすいか**
- **開発フローに溶け込むか**

私たちのチームでは、GitHub Projectsがこれらの条件を満たしていました。

### 前の記事との連携

この記事では、NotionからGitHub Projectsへの**具体的なツール移行**について紹介しました。

**プロセスの変更**（スプリントベースからカンバンへ）については、前回の記事「[AI時代のチーム運営 - スプリントベースからカンバンへの移行](リンクを挿入)」をご覧ください。

両方の変更は、**AIファーストチームへの変革**という同じ文脈で行われています。

---

この記事が、同じような課題に取り組むチームの参考になれば幸いです。もし質問やフィードバックがあれば、ぜひコメントやXでお気軽に声をかけてください。
