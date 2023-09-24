---
title: "Trac 1.6 を Docker で建てる"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["trac"]
published: true
---

# Trac 1.6 を Docker で建てる

2023-09-24 に [Trac 1.6 がリリース](https://pypi.org/project/Trac/1.6/)されました。ようやく Python3で動作するtracが General Availability になりました。めでたい!!
この記事では、Trac 1.6 を Dockerで建てる方法を紹介します。

## trac とは

[trac](https://trac.edgewall.org/)とは、Python で書かれた 課題管理ツール (Issue Tracker) です。バックエンドDBとして SQLite、PostgreSQL、MySQLを利用できます。バージョン管理の git や svn と連携可能です。ユーザ認証の既定値はフロントに建てたWebサーバの認証を利用しますが、プラグインを導入することで独自管理が可能です。

この記事では以下を選択します:

- バックエンドDB: お手軽に SQLite
- バージョン管理: (設定しない)
- ユーザ認証 AccountManagerPlugin による tracローカルでの管理

## Dockerfile を用意する

以下の内容の Dockerfile という名前のファイルを用意します:

```Dockerfile
FROM python:3.9.18-slim-bullseye

ARG TRAC_DIR=/projects
ARG TRAC_PROJECT_NAME=trac_project
ARG DB_LINK=sqlite:db/trac.db

ENV TRAC_DIR ${TRAC_DIR}

#RUN apt-get update && apt-get install -y --no-install-recommends \
#  git \
#  && rm -rf /var/lib/apt/lists/*

RUN pip install \
        babel>1.3 \
        pygments \
        Trac>1.6 \
        'https://trac-hacks.org/browser/accountmanagerplugin/trunk?rev=18549&format=zip'

RUN mkdir ${TRAC_DIR}
WORKDIR ${TRAC_DIR}

RUN trac-admin ${TRAC_DIR} initenv ${TRAC_PROJECT_NAME} ${DB_LINK} && \
    trac-admin ${TRAC_DIR} config set components trac.web.auth.LoginModule disabled && \
    trac-admin ${TRAC_DIR} config set account-manager password_store HtPasswdStore && \
    trac-admin ${TRAC_DIR} config set account-manager htpasswd_hash_type sha256 && \
    trac-admin ${TRAC_DIR} config set account-manager htpasswd_file /trac.htpasswd && \
    trac-admin ${TRAC_DIR} config set account-manager reset_password false && \
    trac-admin ${TRAC_DIR} config set components acct_mgr.admin.* enabled && \
    trac-admin ${TRAC_DIR} config set components acct_mgr.api.* enabled && \
    trac-admin ${TRAC_DIR} config set components acct_mgr.db.sessionstore disabled && \
    trac-admin ${TRAC_DIR} config set components acct_mgr.htfile.htdigeststore disabled && \
    trac-admin ${TRAC_DIR} config set components acct_mgr.htfile.htpasswdstore enabled && \
    trac-admin ${TRAC_DIR} config set components acct_mgr.http.* disabled && \
    trac-admin ${TRAC_DIR} config set components acct_mgr.notification.* enabled && \
    trac-admin ${TRAC_DIR} config set components acct_mgr.pwhash.* disabled && \
    trac-admin ${TRAC_DIR} config set components acct_mgr.register.* enabled && \
    trac-admin ${TRAC_DIR} config set components acct_mgr.svnserve.svnservepasswordstore disabled && \
    trac-admin ${TRAC_DIR} config set components acct_mgr.web_ui.* enabled && \
    trac-admin ${TRAC_DIR} config set components acct_mgr.web_ui.resetpwstore disabled && \
    trac-admin ${TRAC_DIR} config set components trac.web.auth.loginmodule disabled

#RUN mkdir -p ${TRAC_DIR}/git
#RUN cd ${TRAC_DIR}/git; git init

CMD tracd ${TRAC_DIR}
```

## Dockerイメージを作成する

Dockerfileがあるフォルダで、以下のコマンドを実行してDockerイメージを作成します:

```shell-session
$ docker build -t trac:1.6 .
$ # 確認する
$ docker images
(trac    1.6 という行があれば成功)
```

## 試しに実行する

```shell-session
$ docker run -it -p 8080:80 trac:1.6
   :
Serving on 0.0.0.0:80 view at http://127.0.0.1:80/
Using HTTP/1.1 protocol version
```

ブラウザから http://localhost:8080/projects を開き、`Welcome to Trac` という画面が出てくればここまではOK。

### AccountManagerを初期セットアップ

右上の「ログイン」をクリックしてAccountManagerを初期セットアップする:

- Step 1 の Authentication Options では、Authentication Front-end に Use a HTML login form を選ぶ。
- それ以外は、Step5までとりあえず既定値でもOK。
- step6で管理者のユーザ名とパスワードを設定する。(このユーザだけが管理者特権を持つ)
- step6の画面下部で「変更を適用」をクリックし、保存してトップ画面に戻る。
- トップ画面上部右の「ログアウト」→「ログイン」し、step6で作成したユーザでログインする。

----

Enjoy!