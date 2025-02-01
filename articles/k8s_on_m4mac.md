---
title: "Mac Mini M4 に Kubernetes 環境を構築する"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes", "macos" ]
published: false
---

## tl;dr

Docker Desktop for mac をインストールしてもDockerが起動しなくて迷ったので、通れた道をメモしておくことにする。

1. homebrew 4.4.19 https://brew.sh/ja/
1. brew install docker
1. brew install docker-compose
1. brew install docker-buildx
1. vi ~/.docker/config.json
```"cliPluginsExtraDirs": [
		"/opt/homebrew/lib/docker/cli-plugins"
	]```
1. brew install colima
1. brew services start colima
1. brew install orbstack
1. brew install kubectl
1. kubectl version --client
```
Client Version: v1.32.1
Kustomize Version: v5.5.0
```
1. kubectl cluster-info
```

```