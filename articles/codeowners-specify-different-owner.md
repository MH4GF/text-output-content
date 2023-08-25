---
title: "GitHubのCODEOWNERSで一部サブディレクトリだけ別のオーナーを指定する"
emoji: "🎧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [github]
publication_name: route06
published: true
---

## 結論

以下のように指定することで実現できます。

```codeowners
docs/ @docs-owners
docs/another-dir/
docs/another-dir/ @another-team
```

## 背景

GitHub のコードオーナーは、プルリクエストを作成した際に自動的にレビュワーをリクエストすることができる機能です。 `CODEOWNERS` という名前のファイルをリポジトリに配置し、.gitignore と似たような構文でディレクトリと対応するオーナーを指定します。詳しくは以下のドキュメントに記載されています。

https://docs.github.com/ja/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners

CODEOWNERS を利用する上で、「特定のディレクトリのオーナーを指定しつつ、一部サブディレクトリだけ別のオーナーを指定したい」という状況がありました。

```
docs/foo-dir/file.txt      ... @docs-owners
docs/bar-dir/file.txt      ... @docs-owners
docs/another-dir/file.txt  ... @another-team
```

`docs/` ディレクトリは基本的には `@docs-owners` がオーナーですが、 `docs/another-dir/` だけ `@another-team` がオーナーになるようにしたいです。

## 解決方法

以下のように指定することで実現できました。

```codeowners
docs/ @docs-owners
docs/another-dir/
docs/another-dir/ @another-team
```

肝となるのは**一度空の指定を挟むこと**でした。これは気づけなかった...

ドキュメントには以下のように記載されており、空の指定を挟むとそのサブディレクトリのオーナーを取り消すことができるようです。

> \# In this example, @octocat owns any file in the `/apps`  
> \# directory in the root of your repository except for the `/apps/github`  
> \# subdirectory, as its owners are left empty.   
> /apps/ @octocat  
> /apps/github

今回やりたかったことはオーナーを取り消しつつ別のオーナーを指定することですが、その下に該当するチームを指定することで解決できました。  
こちらの情報は [@sinsoku_listy](https://twitter.com/sinsoku_listy)さんに情報共有いただきました、ありがとうございました！  

https://twitter.com/sinsoku_listy/status/1691751626770718801?s=20

---

上記の指定に至るまでにいくつかデバッグとして試しましたが、以下のように指定した場合うまくいきませんでした。

```codeowners
docs/ @docs-owners
docs/another-dir/ @another-team
```

`docs/another-dir/file.txt` への変更の場合、PR のレビュワーには `@docs-owner` と `@another-team` の両方が指定されてしまいます。

```codeowners
docs/ @docs-owners
docs/another-dir/ @another-team
docs/another-dir/
```

`docs/another-dir/file.txt` への変更の場合、PR のレビュワーには何も指定されなくなってしまいます。
