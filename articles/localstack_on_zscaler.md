---
title: "ZScaler管理下でlocalstackを使う"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["localstack", "zscaler" ]
published: true
---

Dockerイメージ `localstack/localstack:latest` には zscalerの証明書が入っておらず、zscaler管理下では起動してもエラーになる。
zscaler入りのイメージを作ることで、zscaler管理下でのエラーを回避できる。

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
