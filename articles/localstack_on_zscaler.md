---
title: "ZScaler管理下でlocalstackを使う"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["localstack", "zscaler" ]
published: true
---

[localstack](https://localstack.cloud/) の Dockerイメージ [localstack/localstack:latest](https://hub.docker.com/r/localstack/localstack) には zscalerの証明書が入っておらず、zscaler管理下では起動するとSSLエラーになる。
zscaler入りのイメージを作ることで、zscaler管理下でのSSLエラーを回避できる。

# 作り方

以下の内容のDockerfileを用意する:

```Dockerfile
# Dockerfile
FROM localstack/localstack:latest

COPY zscaler.cer /usr/share/ca-certificates/
# debian
RUN echo zscaler.cer >> /etc/ca-certificates.conf
RUN update-ca-certificates
# certifi
RUN cat /usr/share/ca-certificates/zscaler.cer >> /opt/code/localstack/.venv/lib/python3.10/site-packages/certifi/cacert.pem
```

ビルドする:

```shell
docker build --tags localstack/localstack:zscaler .
```
