---
title: "GitHub Apps / GitHub Actionsを使って別のリポジトリにファイルをコピーするPRを作成する"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [github,githubactions]
published: false
---

# やりたいこと

GraphQLのスキーマファイルの更新があった際に別のリポジトリにファイルを同期させたい。

# 結論

以下のワークフローファイルでファイルのコピー・別リポジトリへのブランチの作成・コミット・プッシュ・PRの作成ができます。

```yaml
name: copy GraphQL schema to another-repo

on:
  push:
    branches:
      - main
    paths:
      - "app/graphql/schema.graphql"
  workflow_dispatch:

jobs:
  copy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      - name: Generate token
        id: generate_token
        uses: tibdex/github-app-token@v1
        with:
          app_id: ${{ secrets.MY_GITHUB_APP_ID }}
          private_key: ${{ secrets.MY_GITHUB_APP_PRIVATE_KEY }}
      - name: Setup commit description
        env:
          RUN_URL: "${{ github.event.repository.html_url }}/actions/runs/${{ github.run_id }}"
          HEAD_COMMIT_URL: "${{ github.event.repository.html_url }}/commit/${{ github.event.after || github.sha }}"
        id: setup_description
        run: |
          DESCRIPTION=$(cat << EOF
          update GraphQL Schema file 🚀

          created from ${{ env.RUN_URL }}
          latest commit: ${{ env.HEAD_COMMIT_URL }}
          )
          DESCRIPTION="${DESCRIPTION//$'\n'/'%0A'}"
          echo "::set-output name=content::$DESCRIPTION"
      - name: Copy GraphQL Schema file 
        uses: dmnemec/copy_file_to_another_repo_action@v1.1.1
        env:
          API_TOKEN_GITHUB: ${{ steps.generate_token.outputs.token }}
        with:
          source_file: "app/graphql/schema.graphql"
          destination_repo: "MyOrg/another-repo"
          destination_folder: "src/apis/graphql"
          destination_branch: "main"
          destination_branch_create: "feature/update-graphql-schema"
          user_email: "github-actions[bot]@users.noreply.github.com"
          user_name: "github-actions[bot]"
          commit_message: ${{ steps.setup_description.outputs.content }}
      - name: create Pull Request to another-repo
        run: |
          gh pr create \
            --title "update GraphQL Schema file " \
            --body "${{ steps.setup_description.outputs.content }}" \
            --repo MyOrg/another-repo \
            --base main \
            --head feature/update-graphql-schema \
            --reviewer "${{ github.event.head_commit.committer.username || github.triggering_actor }}"
        env:
          GH_TOKEN: ${{ steps.generate_token.outputs.token }}
```

以降のセクションで解説していきます。

# on:

```yaml
on:
  push:
    branches:
      - main
    paths:
      - "app/graphql/schema.graphql"
  workflow_dispatch:
```

今回はmainブランチのpush && 特定ファイルの変更差分がある場合にのみワークフローを発火させています。
何かあった際に手動実行できるようworkflow_dispatchも用意しています。

# steps: Generate token

GitHub Apps経由でトークンを生成するステップです。まずなぜGitHub Appsを利用しているか？について触れます。

## GitHub Actionsで別リポジトリを操作するための権限

今回はGitHub Actionsでファイルの別リポジトリへのコピーを行います。
自身のリポジトリだけでは処理が完結しないため、GitHub ActionsデフォルトのGITHUB_TOKENでは権限が不十分であり、Private Access Token(以下PAT)など何らかのトークンを用意する必要があります。

しかしPATでは以下のような問題があり、個人用途であればいいものの、Organizationで管理しているリポジトリでは使うべきではありません。

- 開発者が退職した際に障害へ発展する可能性がある・組織として管理できない
- PATが流出した際の停止処理が開発者本人にしかできない
- トークンの使用期限が無期限、もしくはある程度長期間になってしまうことが多い
  - 期限を1日など短く設定できるが、毎日設定し直すことは現実的に難しい
- トークンの権限設定でrepoの権限を追加した場合、ユーザーが閲覧可能な全てのリポジトリが操作可能になり、リポジトリ単位の可視性の設定ができない

その代替手段として用意されているのがGitHub Appsです。学習コストが一定発生するのがデメリットですが、以下のようなメリットがあります。

- 特定の個人に依存せず組織として管理することができる
- トークンが短期間（1時間）で失効し、都度生成される
- リポジトリ単位で可視性を設定できるなど、権限管理を柔軟に行える
- トークンを生成するためのprivate keyの失効やAppの権限範囲の調整などをOrganizationで管理することができる

GitHub Appsの設定方法については解説記事が多く存在しているためここでは深く触れませんが、PoCとして検証したスクラップがあるためそちらもよければ参考にしてください。

https://zenn.dev/mh4gf/scraps/3604049b2c53f9

また、今回のワークフローをそのまま利用した場合に必要なGitHub Appsの権限は以下となります。

- Contents: Read and Write
  - コピー先リポジトリのgit pushに必要
- Pull Requests: Read and Write
  - コピー先リポジトリへのPull Requestの作成に必要
- アプリをOrganizationにインストールし、その際にコピー元・コピー先リポジトリを選択

## tibdex/github-app-token 

```yaml
- name: Generate token
  id: generate_token
  uses: tibdex/github-app-token@v1
  with:
    app_id: ${{ secrets.MY_GITHUB_APP_ID }}
    private_key: ${{ secrets.MY_GITHUB_APP_PRIVATE_KEY }}
```

https://github.com/marketplace/actions/github-app-token を利用してトークンを取得します。ここで取得したトークンは後続のステップで `${{ steps.generate_token.outputs.token }}` のように利用できます。

# Steps: Setup commit description 
```yaml
- name: Setup commit description
  env:
    RUN_URL: "${{ github.event.repository.html_url }}/actions/runs/${{ github.run_id }}"
    HEAD_COMMIT_URL: "${{ github.event.repository.html_url }}/commit/${{ github.event.after || github.sha }}"
  id: setup_description
  run: |
    DESCRIPTION=$(cat << EOF
    update GraphQL Schema file 🚀

    created from ${{ env.RUN_URL }}
    latest commit: ${{ env.HEAD_COMMIT_URL }}
    )
    DESCRIPTION="${DESCRIPTION//$'\n'/'%0A'}"
    echo "::set-output name=content::$DESCRIPTION"
```

PRがどのようなコンテキストで作成されたのか？を理解するために有用な情報として、コミットメッセージとしてGitHub ActionsのRunページのURLとファイル差分が追加されたコミットのURLを追加しています。
ワークフローの発火イベントによって `github.event` に入っているオブジェクトが変わるため注意した方が良いです。workflow_dispatchでは `github.event.after` はないためフォールバック先を定義しています。

GitHub Actionsのset-outputで複数行の文字列を渡すにはワークアラウンド的な対応として改行文字の置換が必要でした。こちらを参考にしています。

https://trstringer.com/github-actions-multiline-strings/

# Steps: Copy GraphQL Schema file

```yaml
- name: Copy GraphQL Schema file 
  uses: dmnemec/copy_file_to_another_repo_action@v1.1.1
  env:
    API_TOKEN_GITHUB: ${{ steps.generate_token.outputs.token }}
  with:
    source_file: "app/graphql/schema.graphql"
    destination_repo: "MyOrg/another-repo"
    destination_folder: "src/apis/graphql"
    destination_branch: "main"
    destination_branch_create: "feature/update-graphql-schema"
    user_email: "github-actions[bot]@users.noreply.github.com"
    user_name: "github-actions[bot]"
    commit_message: ${{ steps.setup_description.outputs.content }}
```

https://github.com/marketplace/actions/push-a-file-to-another-repository を利用して特定ファイルを別のリポジトリにコピー・git branchの作成・commit・pushを行います。こちらの記事も参考になりました。

https://zenn.dev/seya/articles/96ea2e7d81d0be

# Steps: create Pull Request to another-repo

```yaml
- name: create Pull Request to another-repo
  run: |
    gh pr create \
      --title "update GraphQL Schema file " \
      --body "${{ steps.setup_description.outputs.content }}" \
      --repo MyOrg/another-repo \
      --base main \
      --head feature/update-graphql-schema \
      --reviewer "${{ github.event.head_commit.committer.username || github.triggering_actor }}"
  env:
    GH_TOKEN: ${{ steps.generate_token.outputs.token }}
```

GitHub CLIを使ってPRを作成します。
レビュワーとして、変更差分を追加したコミッターをアサインしています。これは開発プロセスでの運用として、生成したPRのオーナーシップを明示するために関係がありそうな人をレビュワーに追加したくチームで相談し暫定で追加しました。今後より適任なメンバーが選出できる方法が見つかれば変わるかもしれません。
このレビュワーアサインのように、GitHub ActionsとGitHub CLIの組み合わせで簡単に設定を追加できるのは良いですね。

# 終わりに

今回の記事ではGitHub AppsとGitHub Actionsを使って、ファイルを別リポジトリにコピーする処理を自動化する方法を紹介しました。
個人的にはGitHub Appsを使うことでPATを使わずに安全に操作できるのが嬉しいです。GitHub Actionsもエコシステムが大きくなり使い勝手がどんどん良くなっています。今後も色々な場面で活用していきたいと思います。
