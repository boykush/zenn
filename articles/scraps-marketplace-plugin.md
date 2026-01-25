---
title: "CLIツール×Pluginsで始める、Claude Code駆動のナレッジ管理"
emoji: "🔌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["claude", "claudecode", "cli", "githubactions", "mcp"]
published: true
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

# プラグインとSkills紹介

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

環境変数でMCPサーバーを起動するパスを指定することで、別リポジトリにおいているマークダウンファイル群（Scraps Project）へ容易にアクセスが可能です。

https://boykush.github.io/scraps/scraps/mcp-tools.reference.html

## scraps-writerプラグイン

AIによるインテリジェントなドキュメント作成を支援するプラグインです。MCPツールと組み合わせて、既存のナレッジベースと連携したScrap（マークダウンファイル）作成ができます。

Scraps公式Docに書いているようなWikiリンクシンタックスをSkillsとして理解しています。

https://github.com/boykush/scraps/tree/main/plugins/scraps-writer

マークダウンファイル作成時の最大行数はプロンプトとして重要であったため、Skillsの引数として任意で受け取るようにしました。

### `/add-scrap [title] [max-lines]`　Skill

任意のトピックで新しいマークダウンファイルを作成するスキルです。以下を自動で行います。

- Web検索を通じたトピック調査
- 関連する既存タグの識別
- Wikiリンク用の関連Scrap検索
- 既存Scrapへのバックリンク追加の提案

知らないトピックをキャッチアップしながら、マークダウンファイルへ吐き出すことができます。

### `/web-to-scrap [url] [max-lines]`　Skill

Web記事をマークダウンファイルに変換するスキルです。

- 記事を取得して要約を生成
- OGPカード表示用のソースリンクを自動追加
- タグとWikiリンクで既存ナレッジベースに接続

こちらはRSSフィードに流れてくるWeb記事を要約して吐き出すようなケースをサポートします。

# 実践例: IssueベースScrap追加ワークフロー

ここからは実際にこれらのプラグインを活用したワークフローの例として、GitHub Issueをトリガーに `/add-scrap` スキルでScrapを自動生成する仕組みを紹介します。

Scrapsを用いて私自身のナレッジを管理している以下のリポジトリで実践しました。

https://github.com/boykush/wiki

## ワークフローの実装

GitHub ActionsとClaude Code Actionを組み合わせて、Issueが作成されたら自動的にScrapを生成してPRを作成するワークフローを実装しました。

```yaml
name: Create Scrap from Issue
on:
  issues:
    types: [opened, reopened]

jobs:
  create-scrap:
    if: github.event.issue.user.login == 'boykush'
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Setup mise
        uses: jdx/mise-action@v2

      - name: Install scraps
        run: mise install cargo:scraps

      - name: Run Claude with add-scrap skill
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          plugin_marketplaces: |
            https://github.com/boykush/scraps.git
          plugins: |
            scraps-writer@scraps-claude-code-plugins
          settings: ".claude/settings.json"
          prompt: |
            /add-scrap ${{ github.event.issue.title }} max-lines=15

            追加情報:
            ${{ github.event.issue.body }}

      - name: Create Pull Request
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git remote set-url origin "https://x-access-token:${GITHUB_TOKEN}@github.com/${{ github.repository }}.git"

          BRANCH_NAME="add-scrap/issue-${{ github.event.issue.number }}-${{ github.run_id }}"
          git checkout -b "$BRANCH_NAME"

          git add scraps/
          git commit -m "Add scrap: ${{ github.event.issue.title }}"
          git push origin "$BRANCH_NAME"

          gh pr create \
            --title "Add scrap: ${{ github.event.issue.title }}" \
            --body "Closes #${{ github.event.issue.number }}" \
            --base main
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

ワークフローのポイントは以下です。

- scraps Marketplaceからscraps-writerプラグインを読み込み
- Issueのタイトルを `/add-scrap` Skillsに渡して実行
- 生成されたScrapファイルを含むPRを自動作成

## 実際の動作例

実際にIssueを作成してワークフローが動作した例です。

例として「Platform Engineering Maturity Model」を作成すると、自動的に以下のようなファイルのPRが生成されました。

最大行数に加えて、内容のフォーマットは `CLAUDE.md` にて指示するのが良いと思います。余談として[Scraps自体にTemplate機能はある](https://boykush.github.io/scraps/scraps/use-templates.how-to.html)のですが、LLMによって有用性が減ってきたのでDeprecatedにするかもしれません。

```md
## PEMM

#[[Platform Engineering]] #[[Cloud Native]] #[[Team Organization]]

[[CNCF]]が提供する[[Platform Engineering]]の成熟度を評価するフレームワーク

5つの評価軸で組織のプラットフォーム進捗を測定する

- Investment（投資）
- Adoption（採用）
- Interfaces（インターフェース）
- Operations（運用）
- Measurement（測定）

2026年までに大規模ソフトウェア組織の80%がプラットフォームチームを持つとGartnerが予測しており、AI統合、測定実践、開発者体験の向上が重要な要素となる

<https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/>
```

以下、PRリンクです

https://github.com/boykush/wiki/pull/120

[Actionsの実行ログ](https://github.com/boykush/wiki/actions/runs/21327834839)を見ても、実際にMCPサーバーを呼び出しながらSkillsによってファイルが書き出されていることを把握することができます。

これで後でキャッチアップしたいトピックをIssueにメモすれば、自動でPRが作成されキャッチアップが済んだらPRマージ、のような自動化を実現することができました。

# まとめ

実践例について、途中まではRSSフィードから自動でWeb記事を読み取り、 `/web-to-scrap` SkillsでPR作成する自動化を考えていたのですが、少々複雑で大規模になりそうだったため今回の内容でスモールスタートしました。

とはいえ普段から行っていたキャッチアップメモのIssueを生かして自動化フローを組めたのはよかったです。

Claude CodeのPluginsやSkillsについて、自身で手を動かして理解を深めることができました👏
