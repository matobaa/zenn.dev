---
title: "Mac Mini M4 に Kubernetes 環境を構築する"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes", "macos" ]
published: true
---

## tl;dr

新しく手に入れたM4MacでKubernetesを動かしたかった。

Docker Desktop for mac をインストールしてもDockerが起動しなかったりして道に迷ったので、通れた道をメモしておくことにする。

## やったこと

1. homebrew 4.4.19 https://brew.sh/ja/
1. brew install orbstack
1. FinderからOrbstackを起動して　Kubernetes を選択

## 以下、一回やったけど一切不要だったので戻した

1. brew install docker
1. brew install docker-compose
1. brew install docker-buildx
1. vi ~/.docker/config.json で ` "cliPluginsExtraDirs": [ "/opt/homebrew/lib/docker/cli-plugins" ]` を追記
1. brew install colima
1. brew services start colima
1. brew install kubectl
