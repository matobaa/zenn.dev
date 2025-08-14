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
openssl ecparam -name prime256v1 -genkey -noout -out ca.key
openssl req -new -key ca.key -subj "/CN=kubernetes" -out ca.csr
openssl x509 -req -in ca.csr -signkey ca.key -days 365 -out ca.cer

for i in admin node-0 node-1 node-2 kube-proxy kube-scheduler kube-controller-manager kube-api-server service-account; do
  openssl ecparam -name prime256v1 -genkey -noout -out ${i}.key
  openssl req -new -key ${i}.key -subj /CN=${i} -out ${i}.csr
  openssl x509 -req -in ${i}.csr -CA ca.cer -CAkey ca.key -days 365 -out ${i}.cer
done

openssl x509 -req -in kube-api-server.csr -CA ca.cer -CAkey ca.key -days 365 -out kube-api-server.cer \
-extfile <(echo subjectAltName = DNS:control-0.internal, DNS:control-1.internal, DNS:control-2.internal)

exit
```

: openssl genpkey -algorithm ed25519 -out ${i}.key ->
: [E0814 00:49:51.957412    6531 run.go:72] "command failed" err="failed to build token generator: unknown private key type ed25519.PrivateKey, must be *rsa.PrivateKey, *ecdsa.PrivateKey, or jose.OpaqueSigner"


## Controller Node に必要なファイルをコピーする

```shell-session:m4mac
container exec jumpbox tar cf - -C /root . | container exec -i control-0 tar xf - -C /root
container exec jumpbox tar cf - -C /root . | container exec -i control-1 tar xf - -C /root
container exec jumpbox tar cf - -C /root . | container exec -i control-2 tar xf - -C /root
```

## Controller Node に　etcdをインストールして起動する

```shell-session:control-N
apt-get update && apt-get install -y curl

curl https://storage.googleapis.com/etcd/v3.6.4/etcd-v3.6.4-linux-arm64.tar.gz | tar xzf -
install etcd-v3.6.4-linux-arm64/etcd* /usr/local/bin

cd
etcd --name=$(hostname) \
--peer-key-file /root/kube-api-server.key \
--peer-cert-file /root/kube-api-server.cer \
--peer-trusted-ca-file /root/ca.cer \
--peer-client-cert-auth \
--key-file /root/kube-api-server.key \
--cert-file /root/kube-api-server.cer \
--trusted-ca-file /root/ca.cer \
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
--cert /root/admin.cer \
--cacert /root/ca.cer \
--endpoints=https://$(hostname).internal:2379

etcdctl get mykey \
--key /root/admin.key \
--cert /root/admin.cer \
--cacert /root/ca.cer \
--endpoints=https://$(hostname).internal:2379

etcdctl member list \
--key /root/admin.key \
--cert /root/admin.cer \
--cacert /root/ca.cer \
--endpoints=https://$(hostname).internal:2379

exit
```

## Kubernetesコントロールプレーンのプロビジョニング


```shell-session:control-N

mkdir -p /etc/kubernetes/config
apt-get install wget
: https://github.com/kubernetes/kubernetes
: https://kubernetes.io/releases/download/#binaries

wget -q --show-progress \
https://dl.k8s.io/v1.33.3/bin/linux/arm64/kube-apiserver \
https://dl.k8s.io/v1.33.3/bin/linux/arm64/kube-controller-manager \
https://dl.k8s.io/v1.33.3/bin/linux/arm64/kube-scheduler \
https://dl.k8s.io/v1.33.3/bin/linux/arm64/kubectl

install -m 755 kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin


kube-apiserver \
--service-cluster-ip-range 192.168.64.0/24 \
--service-account-key-file /root/service-account.key \
--service-account-issuer /root/ca.cer \
--service-account-signing-key-file /root/service-account.key \
--etcd-servers https://control-0.internal:2379,https://control-1.internal:2379,https://control-2.internal:2379 \
--etcd-keyfile /root/kube-api-server.key \
--etcd-certfile /root/kube-api-server.cer \
--etcd-cafile /root/ca.cer \
--tls-cert-file=/root/kube-api-server.cer \
--tls-private-key-file=/root/kube-api-server.key \
&

```

### kubectl用の設定

```

kubectl config set-cluster hardway \
  --server=https://control-0.internal:6443 \
  --certificate-authority=/root/ca.cer \
  --embed-certs=true


kubectl config set-context default \
  --cluster=hardway \
  --user=admin

kubectl config use-context default 

kubectl config set-credentials admin \
  --client-certificate=/root/admin.cer \
  --client-key=/root/admin.key \
  --embed-certs=true

```

### ところで　Docker in container は動くの??


https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository　に従って


```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

service docker start

tail /var/log/docker.log
failed to start daemon: Error initializing network controller: error obtaining controller instance: failed to register "bridge" driver: failed to create NAT chain DOCKER: iptables failed: iptables --wait -t nat -N DOCKER: iptables v1.8.10 (nf_tables): Could not fetch rule set generation id: Invalid argument
 (exit status 4)

root@control-1:/# iptables -t nat -N DOCKER
iptables v1.8.10 (nf_tables): Could not fetch rule set generation id: Invalid argument

https://marendasoft.eu/iptables-v1-8-4-nf_tables-could-not-fetch-rule-set-generation-id-invalid-argument/ に従って

apt install iptables
 	update-alternatives --set iptables /usr/sbin/iptables-legacy
 	update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
        apt-cache policy nftables

service docker start

docker ps

docker run hello-world

動いた

##

