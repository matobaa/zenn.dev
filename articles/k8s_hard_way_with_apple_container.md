---
title: "kubernetes_the_hard_way with apple/container"
emoji: "🐕"
type: "tech"
topics: ["kubeernetes", "kubernetes-the-hard-way", "apple/container" ]
published: false
---

### この記事はなに

[kubernetes the hard way](https://github.com/kelseyhightower/kubernetes-the-hard-way) を [apple/container](https://github.com/apple/container) でやってみた記録です。

#### NOTE

- この記事は作成中です。
- ライセンスは CC-BY-NC-SA-4.0 とします。
- コマンド例のうち、`m4mac%`で始まる行は macのzshで実行するもの、それ以外で始まる行は containerから起動した仮想マシン内で実行するものです。

## 1. 準備

まず前提として、apple/containerでネットワークを意図通り設定するにはMacOS26beta以降が必要です(MacOS15.x以前だとコンテナ間の通信が通りません)。

[apple/container](https://github.com/apple/container)の最新版(執筆時現在0.3.0)の[container-0.3.0-installer-signed.pkg](https://github.com/apple/container/releases/tag/0.3.0)をダウンロード・実行してインストールする。

インストール後、以下のようにapple/containerを起動しておく:
```shell-session
m4mac% container --version 
container CLI version 0.3.0 (build: release, commit: 3fcf647)

m4mac% container system start
Verifying apiserver is running...
Installing base container filesystem...
No default kernel configured.                                                              
Install the recommended default kernel from [https://github.com/kata-containers/kata-containers/releases/download/3.17.0/kata-static-3.17.0-arm64.tar.xz]? [Y/n]: Y
Installing kernel...

m4mac% container system status
Verifying apiserver is running...
apiserver is running

m4mac% container system dns create internal
m4mac% container system dns default set internal
```

## 仮想マシンを作る

作業用を1台、コントロールプレーンノードを3台、ワーカーノードを3台起動します。出入りしたいので、シェルではなくsleep infinity を実行しておきます:

```shell-session:m4mac
container run -d --name jumpbox ubuntu sleep infinity
container run -d --name control-0 ubuntu sleep infinity
container run -d --name control-1 ubuntu sleep infinity
container run -d --name control-2 ubuntu sleep infinity
container run -d --name worker-0 ubuntu sleep infinity
container run -d --name worker-1 ubuntu sleep infinity
container run -d --name worker-2 ubuntu sleep infinity

container list | sort -k 5
ID         IMAGE                            OS     ARCH   STATE    ADDR
jumpbox    docker.io/library/ubuntu:latest  linux  arm64  running  192.168.64.2
control-0  docker.io/library/ubuntu:latest  linux  arm64  running  192.168.64.3
control-1  docker.io/library/ubuntu:latest  linux  arm64  running  192.168.64.4
control-2  docker.io/library/ubuntu:latest  linux  arm64  running  192.168.64.5
worker-0   docker.io/library/ubuntu:latest  linux  arm64  running  192.168.64.6
worker-1   docker.io/library/ubuntu:latest  linux  arm64  running  192.168.64.7
worker-2   docker.io/library/ubuntu:latest  linux  arm64  running  192.168.64.8
```

## jumpbox内で、必要な鍵を準備する
```shell-session:m4mac
container exec -it jumpbox bash
```
```shell-session:jumpbox
apt-get update && apt-get -y install wget curl vim openssl 

cd
openssl genpkey -algorithm ed25519  -out ca.key
openssl req -new -key ca.key -subj "/CN=kubernetes" -out ca.csr
openssl x509 -req -in ca.csr -signkey ca.key -days 365 -out ca.crt

for i in admin node-0 node-1 kube-proxy kube-scheduler kube-controller-manager kube-api-server service-accounts; do
  openssl genpkey -algorithm ed25519 -out ${i}.key
  openssl req -new -key ${i}.key -subj /CN=${i} -out ${i}.csr
  openssl x509 -req -in ${i}.csr -CA ca.crt -CAkey ca.key -days 365 -out ${i}.crt
done

<<EOF tee ext.txt
subjectAltName = DNS:control-0.internal, DNS:control-1.internal, DNS:control-2.internal
EOF
openssl x509 -req -in kube-api-server.csr -CA ca.crt -CAkey ca.key -days 365 -extfile ext.txt -out kube-api-server.crt

exit

```

## Controller Node に必要なファイルをコピーする

```shell-session:m4mac
container exec jumpbox tar cf - -C /root . | container exec -i control-0 tar xf - -C /root
container exec jumpbox tar cf - -C /root . | container exec -i control-1 tar xf - -C /root
container exec jumpbox tar cf - -C /root . | container exec -i control-2 tar xf - -C /root
```

## Controller Node に　etcdをインストールして起動する
```shell-session
apt-get update && apt-get install -y curl

curl https://storage.googleapis.com/etcd/v3.6.4/etcd-v3.6.4-linux-arm64.tar.gz | tar xzf -
install etcd-v3.6.4-linux-arm64/etcd* /usr/local/bin

etcd --name=$(hostname) \
--peer-key-file /root/kube-api-server.key \
--peer-cert-file /root/kube-api-server.crt \
--peer-trusted-ca-file /root/ca.crt \
--peer-client-cert-auth \
--key-file /root/kube-api-server.key \
--cert-file /root/kube-api-server.crt \
--trusted-ca-file /root/ca.crt \
--client-cert-auth \
--listen-peer-urls=https://0.0.0.0:2380 \
--listen-client-urls https://0.0.0.0:2379 \
--advertise-client-urls https://$(hostname).internal:2379 \
--initial-advertise-peer-urls=https://$(hostname).internal:2380 \
--data-dir /var/lib/etcd \
--initial-cluster=control-0=https://control-0.internal:2380,control-1=https://control-1.internal:2380,control-2=https://control-2.internal:2380 \
--initial-cluster-state=new \
--initial-cluster-token=etcd-cluster-0 \
&

etcdctl put mykey "I am $(hostname)" \
--key /root/admin.key \
--cert /root/admin.crt \
--cacert /root/ca.crt \
--endpoints=https://$(hostname).internal:2379

etcdctl get mykey \
--key /root/admin.key \
--cert /root/admin.crt \
--cacert /root/ca.crt \
--endpoints=https://$(hostname).internal:2379

etcdctl member list \
--key /root/admin.key \
--cert /root/admin.crt \
--cacert /root/ca.crt \
--endpoints=https://$(hostname).internal:2379
```

## 
