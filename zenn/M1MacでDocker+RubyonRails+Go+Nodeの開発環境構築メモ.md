# はじめに

表題の通り検証してみました。今回の記事ではMacの初期化状態から開発が始められるまでのセットアップ内容と、M1環境において詰まったところを紹介していきます。
2021/4/29に環境構築し、この環境で所属している会社の開発に1ヶ月ほど従事していますが、今のところ特に問題もなく開発できています。今後も継続してM1環境で開発するつもりです。
M1の検証記事はなんぼあってもいいですからね。他の記事との重複を気にせず記載していきます。

# 今回の環境

- Homebrew, Rosetta 2を利用しています。
- RubyのランタイムはDocker for Macを利用しDocker上で動かしています。
- Railsプロジェクトはdocker-composeで立ち上げ、Rails, MySQL, Redis, Sidekiqを動かしています。
- Go/Nodeはanyenvを利用しホストマシンにインストールして動かしています。

# Macのリストア・依存ライブラリやアプリケーションのインストール

自分はMacを定期的に工場出荷状態に戻したい民で、Brewfileやシェルスクリプトで初期セットアップを自動化しています。gistに記載しています。
https://gist.github.com/MH4GF/83cd437393caa30afb3ea48807320f03
パブリックなgistにシェルスクリプトを置いておくことで、Macを初期化した状態でもすぐセットアップ処理が始められて便利です。
Dropboxにsshキーや設定ファイルを置いているため僕以外の環境では動かせませんが、おそらくどのような処理をしているのかは理解してもらえるかなと思っています。
シェルスクリプトの処理内容としてはざっくり以下です。

1. Xcode, brewをインストール
2. Dropboxをインストールし、sshキーやgitの設定
3. gistに置いているBrewfileを取得し各種ソフトウェアのインストール
4. anyenvを利用してNode/Goのインストール
5. defaultsコマンドなどMacの設定変更

特筆すべき内容をそれぞれピックアップしていきます。

## Homebrewのインストール

Homebrewについてはバージョン3.0.0で正式にApple Siliconがサポートされています。Intel版との違いがあるとすれば別のディレクトリにインストールされているためパスを通す必要がある程度です。

```
echo "Homebrewがなければインストール"
if [ ! -x "`which brew`" ]; then
  echo "ないのでインストール始めます"
  /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
  
  # M1の場合pathを通す必要がある
  if [ $ARCH == "arm64" ]; then
    echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
    eval "$(/opt/homebrew/bin/brew shellenv)"
  fi
  brew update
  brew upgrade
  brew -v
fi
brew doctor
```

## Brewfileを利用した各種ソフトウェアのインストール

```
echo "BrewfileをGistから取得する"
curl -fsSL https://gist.githubusercontent.com/MH4GF/c945f8e7654dcf1db7a2928885068167/raw > ~/.Brewfile

echo "brewで各ソフトウェアをインストール"
brew bundle --global
```

`brew bundle` コマンドを使って各種ソフトウェアやアプリケーションをインストールします。
Brewfileの中身は現在このようになっています。

```
tap 'homebrew/cask'
tap 'homebrew/cask-versions'
tap 'homebrew/core'
tap 'homebrew/bundle'
tap 'mh4gf/cleanup'

brew 'git'
brew 'tig'
brew 'ghq'
brew 'peco'
brew 'jq'
brew 'awscli'
brew 'versent/taps/saml2aws'
brew 'anyenv'
brew 'MH4GF/cleanup/cleanup'

cask 'slack'
cask 'clipy'
cask 'karabiner-elements'
cask 'skitch'
cask 'google-chrome'
cask 'iterm2'
cask 'docker'
cask 'google-drive-file-stream'
cask 'adobe-creative-cloud'
cask 'bettertouchtool'
cask '1password'
cask 'intellij-idea'
cask 'goland'
cask 'rubymine'
cask 'webstorm'
cask 'figma'
cask 'deepl'

mas 'LINE', id: 539_883_307
```

まずbrew経由でのインストールで失敗したのはsaml2awsのみでした。
とはいえど後述するGoのセットアップが終わった後リポジトリをcloneしてきて `make install` を打てば解決しました。
このように、ある程度一般的なソフトウェアはM1環境でもインストール自体はできるものと思っていいのかなと思います。僕もここでかなり詰まるだろうと想定していたので意外でした。

### ソフトウェアの実行やRosetta2周り

上記でインストールしたソフトウェアのうち、起動・実行に失敗したものもありませんでした。もちろんIntel版しか対応していないソフトウェアについては起動時にRosetta2をインストールする必要はあります。

また、ソフトウェアの公式ホームページ等ではApple Silicon版の対応バイナリがあるのに、brew経由でインストールした場合Intel版がインストールされてしまった、ということもよくあります。
これはHomebrewやHomebrew-caskがApple Silicon版への対応ができていないかもしれません。今回RubyMineの修正対応でPull Requestを出してみましたがすぐにマージされました。修正はそこまで難しくなかったため、使っているソフトウェアがbrewで対応していなかった場合はぜひPull Requestを出してみてください。

https://github.com/Homebrew/homebrew-cask/pull/105117

## Node/Goのランタイムのインストール
自分はNode/Goについてはanyenvを利用してMacに直接ランタイムをインストールして利用しています。こちらについてはIntel版のMacと何も変わらず環境構築と実行ができました。

```
echo "XXenv系のインストール"
if [ ! -x "`which node`" ]; then
  echo "nodenvのインストール"
  anyenv install nodenv
  exec $SHELL -l
  nodenv install $NODENV_VERSION
  nodenv global $NODENV_VERSION
fi
node -v

if [ ! -x "`which go`" ]; then
  echo "goenvのインストール"
  anyenv install goenv
  exec $SHELL -l
  goenv install $NODENV_VERSION
  goenv global $NODENV_VERSION
  goenv rehash
fi
go version
```

# docker-composeでのRailsプロジェクトの環境構築

RailsプロジェクトについてはIntel版Macであってもdocker-composeで環境構築することが多いです。今回もdocker-composeで立ち上げます。起動するコンテナは以下で、ある程度一般的な構成だと思われます。

- pumaで起動するRailsアプリケーション
- MySQL5.7
- Redis5.0
- Sidekiq

ここからは立ち上げ途中で詰まった箇所についてピックアップしていきます。

## mysqlイメージのpullに失敗

```
Pulling mysql (mysql:5.7)...
5.7: Pulling from library/mysql
ERROR: no matching manifest for linux/arm64/v8 in the manifest list entries
```
これはDocker公式でも既知の問題であり、[MySQLイメージはarmに対応していないとのこと。](https://docs.docker.com/docker-for-mac/apple-silicon/)


> the mysql image is not available for ARM64. You can work around this issue by using a mariadb image.

こちらについてはdocker-compose.ymlのserviceにて `platform: linux/x86_64` を追加することでpullと起動に成功しました。
https://stackoverflow.com/questions/65456814/docker-apple-silicon-m1-preview-mysql-no-matching-manifest-for-linux-arm64-v8

## rails server起動後、アクセス時に symbol lookup errorが出る

```
app_1      | Started GET "/login" for 172.18.0.1 at 2021-05-05 10:10:15 +0900
app_1      |    (2.1ms)  SET NAMES utf8mb4 COLLATE utf8mb4_bin,  @@SESSION.sql_mode = CONCAT(CONCAT(@@sql_mode, ',STRICT_ALL_TABLES'), ',NO_AUTO_VALUE_ON_ZERO'),  @@SESSION.sql_auto_is_null = 0, @@SESSION.wait_timeout = 2147483
app_1      |    (1.4ms)  SELECT `schema_migrations`.`version` FROM `schema_migrations` ORDER BY `schema_migrations`.`version` ASC
app_1      | Processing by SessionsController#new as HTML
app_1      |   Rendering devise/sessions/new.html.slim 
app_1      |   Rendered devise/sessions/new.html.slim (Duration: 9.7ms | Allocations: 9394)
app_1      | puma 3.12.6 (tcp://0.0.0.0:3000) [app]: symbol lookup error: /usr/local/bundle/gems/ffi-1.15.0/lib/ffi_c.so: undefined symbol: pthread_atfork
hogehoge_app_1 exited with code 127
```

こちらは依存ライブラリであるffiのバージョンが1.15.0だと起こるようで、以下のissueのように