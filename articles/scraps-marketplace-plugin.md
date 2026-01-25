---
title: "CLIツール×Pluginsですぐに始める、Claude Code駆動のナレッジ管理"
emoji: "🔌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["claudecode", "cli", "github", "mcp"]
published: false
---

# はじめに

前回、Wikiリンク記法で書いたマークダウンファイルを静的サイトへビルドするCLIツール「Scraps」のMCPサーバー接続について紹介しました。

https://zenn.dev/boykush/articles/1aa8848b23f09a

今回はその続きとして、Claude Code Plugin Marketplaceを使ったプラグインの配布と、Scrapsが提供するプラグイン・スキルについて紹介します。

開発当初はSSGを中心とする機能開発をしていましたが、CLIツールである強みを生かしてMCPサーバーを中心とするAI連携に力を入れています。


# Claude Code Plugin Marketplaceとは

Claude Codeにはプラグインの発見・配布・インストールを行うMarketplaceの仕組みがあります。Marketplaceはプラグインのカタログとして機能し、GitHubリポジトリやGit URL、ローカルパスなどから追加できます。

https://code.claude.com/docs/en/plugin-marketplaces
https://code.claude.com/docs/en/discover-plugins

# ScrapsのMarketplace

ScrapsはClaude Code向けのプラグインをMarketplaceとして提供しています。以下のコマンドでMarketplaceを追加できます。

```bash
/plugin marketplace add boykush/scraps
```

追加後は `/plugin` からプラグインを参照・インストールできます。

https://boykush.github.io/scraps/scraps/install-claude-code-plugin.how-to.html

# プラグインとSkill紹介

ScrapsのMarketplaceでは2つのプラグインを提供しています。

## mcp-serverプラグイン

前回の記事で紹介したMCPサーバー機能をプラグインとして提供しています。

https://github.com/boykush/scraps/tree/main/plugins/mcp-server

MCPツール群でナレッジベースの検索・参照ができます。

- **search_scraps**: タイトル・コンテンツの曖昧検索
- **lookup_scrap_links**: Scrapから貼られている内部リンク一覧
- **lookup_scrap_backlinks**: Scrapへ向けられている内部リンク一覧
- **list_tags**: Tag一覧の取得
- **lookup_tag_backlinks**: Tagへ向けられている内部リンク一覧

https://boykush.github.io/scraps/scraps/mcp-tools.reference.html

## scraps-writerプラグイン

AIによるインテリジェントなドキュメント作成を支援するプラグインです。MCPツールと組み合わせて、既存のナレッジベースと連携したScrap作成ができます。

https://github.com/boykush/scraps/tree/main/plugins/scraps-writer

### `/add-scrap [title] [max-lines]`

任意のトピックで新しいScrapを作成するスキルです。以下を自動で行います。

- Web検索を通じたトピック調査
- 関連する既存タグの識別
- Wikiリンク用の関連Scrap検索
- 既存Scrapへのバックリンク追加の提案

### `/web-to-scrap [url] [max-lines]`

Web記事をScrapに変換するスキルです。

- 記事を取得して要約を生成
- OGPカード表示用のソースリンクを自動追加
- タグとWikiリンクで既存ナレッジベースに接続

# 実践例: IssueベースScrap追加ワークフロー

ここからは実際にこれらのプラグインを活用したワークフローの例として、GitHub Issueをトリガーに `/add-scrap` スキルでScrapを自動生成する仕組みを紹介します。

https://github.com/boykush/wiki

## ワークフローの実装

GitHub ActionsとClaude Code Actionを組み合わせて、Issueが作成されたら自動的にScrapを生成してPRを作成するワークフローを実装しました。

https://github.com/boykush/wiki/pull/118

ワークフローのポイントは以下です。

- `anthropics/claude-code-action@v1` を使用
- scraps Marketplaceからscraps-writerプラグインを読み込み
- Issueのタイトルを `/add-scrap` スキルに渡して実行
- 生成されたScrapファイルを含むPRを自動作成

## 実際の動作例

実際にIssueを作成してワークフローが動作した例です。

https://github.com/boykush/wiki/issues/119

このIssue「Platform Engineering Maturity Model」を作成すると、自動的に以下のPRが生成されました。

https://github.com/boykush/wiki/pull/120

`/add-scrap` スキルがWeb検索でトピックを調査し、既存のナレッジベースから関連タグを選定、Wikiリンクで接続可能なScrapを提案してくれます。

# まとめ

Claude Code Plugin Marketplaceを使うことで、自作ツールの機能をプラグインとして配布・共有できるようになります。Scrapsでは以下を提供しています。

- **mcp-server**: ナレッジベースの検索・参照
- **scraps-writer**: AIによるインテリジェントなScrap作成

GitHub ActionsとClaude Code Actionを組み合わせれば、Issueをトリガーにした自動ドキュメント生成のようなワークフローも実現できます。

前回の記事ではMCPサーバー経由で「自分のナレッジを参照する」ことができましたが、今回のプラグイン・スキルによって「自分のナレッジに追加する」部分も自動化できるようになりました。インプットとアウトプットの両方でLLMと自分の脳が繋がっていく感覚があります。
