---
title: "kubernetes_the_hard_way on apple/container"
emoji: "📦"
type: "tech"
topics: ["kubernetes", "kubernetes-the-hard-way", "apple/container" ]
published: false
---

### 概要

[kubernetes the hard way](https://github.com/kelseyhightower/kubernetes-the-hard-way) を [apple/container](https://github.com/apple/container) で試した記録です。

### 注記

- この記事は作成中です。
- ライセンスは CC-BY-NC-SA-4.0 とします。
- コマンド例のプロンプト `m4mac%` はmacOSのzsh、それ以外はコンテナ内の仮想マシンで実行することを示します。
- [Options for Highly Available Topology](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/) の Stacked etcd topology を採用します。Control Planeの各ノードにetcdを配置し、API-Serverは同じノードのetcdに接続する構成です。

### ファイル配置

証明書などのファイルは、[こちらの記事](https://qiita.com/FY0323/items/6a3b3270888c96ba13d3#%E5%90%84%E7%A8%AE%E8%A8%BC%E6%98%8E%E6%9B%B8%E3%81%AE%E4%BD%9C%E6%88%90)を参考に、以下の場所に配置します。

- `/etc/kubernetes/kubelet.conf`
- `/etc/kubernetes/controller-manager.conf`
- `/etc/kubernetes/scheduler.conf`
- `/etc/kubernetes/admin.conf`
- `/etc/kubernetes/pki/`
  - `ca.crt`, `ca.key`
  - `apiserver.crt`, `apiserver.key`
  - `apiserver-kubelet-client.crt`, `apiserver-kubelet-client.key`
  - `sa.pub`, `sa.key`
  - front proxy用: `front-proxy-ca.crt`, `front-proxy-ca.key`

## 1. 準備

### 前提条件

`apple/container` のネットワークを正しく設定するには、macOS 26 beta以降が必要です。以前のバージョンではコンテナ間通信ができません。

### インストール

[apple/container](https://github.com/apple/container)の最新版（本記事執筆時点では0.3.0）を[ダウンロード](https://github.com/apple/container/releases/tag/0.3.0)し、インストールします。

インストール後、`apple/container`を起動します。

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

## 2. 仮想マシンの作成

作業用1台、コントロールプレーン3台、ワーカー3台の仮想マシンを起動します。コンテナに自由に出入りできるよう、`sleep infinity`で起動し続けます。

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

## 3. 鍵とバイナリの準備

作業用の`jumpbox`コンテナ内で、各ノードで必要となる鍵とバイナリを準備します。

```shell-session:m4mac
container exec -it jumpbox bash
```

```shell-session:jumpbox
# 必要なパッケージをインストール
apt-get update && apt-get -y install wget curl vim openssl 

# バイナリの配置先ディレクトリを作成
cd
mkdir -p /root/control/usr/local/bin
mkdir -p /root/worker/usr/local/bin

# etcdのバイナリをダウンロード・展開
curl https://storage.googleapis.com/etcd/v3.6.4/etcd-v3.6.4-linux-arm64.tar.gz | tar xzf -
install etcd-v3.6.4-linux-arm64/etcd* /root/control/usr/local/bin

# Kubernetesのバイナリをダウンロード
# ref: https://kubernetes.io/releases/download/#binaries
wget -q --show-progress \
https://dl.k8s.io/v1.33.3/bin/linux/arm64/kube-apiserver \
https://dl.k8s.io/v1.33.3/bin/linux/arm64/kube-controller-manager \
https://dl.k8s.io/v1.33.3/bin/linux/arm64/kube-scheduler \
https://dl.k8s.io/v1.33.3/bin/linux/arm64/kubectl \
https://dl.k8s.io/v1.33.3/bin/linux/arm64/kube-proxy \
https://dl.k8s.io/v1.33.3/bin/linux/arm64/kubelet \

# バイナリを所定の場所に配置
install -m 755 kubectl kube-apiserver kube-controller-manager kube-scheduler /root/control/usr/local/bin
install -m 755 kubectl kube-proxy kubelet /root/worker/usr/local/bin/

# PKI(公開鍵基盤)のディレクトリを作成
mkdir -p /etc/kubernetes/pki
cd /etc/kubernetes/pki

# CA(認証局)の鍵と証明書を作成
openssl ecparam -name prime256v1 -genkey -noout -out ca.key
openssl req -new -key ca.key -subj "/CN=kubernetes" -out ca.csr
openssl x509 -req -in ca.csr -signkey ca.key -days 365 -out ca.crt

# 各コンポーネントの鍵と証明書を作成
for i in admin worker-0 worker-1 worker-2 kube-proxy kube-scheduler kube-controller-manager api-server service-account; do
  openssl ecparam -name prime256v1 -genkey -noout -out ${i}.key
  openssl req -new -key ${i}.key -subj /CN=${i} -out ${i}.csr
  openssl x509 -req -in ${i}.csr -CA ca.crt -CAkey ca.key -days 365 -out ${i}.crt
done

# APIサーバーの証明書にSANs(Subject Alternative Names)を追加
openssl x509 -req -in api-server.csr -CA ca.crt -CAkey ca.key -days 365 -out api-server.crt \
-extfile <(echo subjectAltName = DNS:control-0.internal, DNS:control-1.internal, DNS:control-2.internal)

# 不要になったCSR(証明書署名要求)ファイルを削除
rm *.csr

exit
```

> **注:** `ed25519` 鍵を使用すると、API-Server起動時に `unknown private key type ed25519.PrivateKey` エラーが発生しました。

## 4. 各ノードへのファイル配布

`jumpbox`で準備した鍵とバイナリを、各Control PlaneノードとWorkerノードにコピーします。

```shell-session:m4mac
# Control Planeノードへのコピー
container exec jumpbox tar cf - etc/kubernetes/pki -C /root/control usr/local/bin | container exec -i control-0 tar xf -
container exec jumpbox tar cf - etc/kubernetes/pki -C /root/control usr/local/bin | container exec -i control-1 tar xf -
container exec jumpbox tar cf - etc/kubernetes/pki -C /root/control usr/local/bin | container exec -i control-2 tar xf -

# Workerノードへのコピー
container exec jumpbox tar cf - etc/kubernetes/pki -C /root/worker usr/local/bin | container exec -i worker-0 tar xf -
container exec jumpbox tar cf - etc/kubernetes/pki -C /root/worker usr/local/bin | container exec -i worker-1 tar xf -
container exec jumpbox tar cf - etc/kubernetes/pki -C /root/worker usr/local/bin | container exec -i worker-2 tar xf -
```

## 5. Control Planeノードの構築

各Control Planeノード (`control-0`, `control-1`, `control-2`) に入り、etcdとKubernetesコンポーネントを起動します。

### etcdの起動

```shell-session:control-N
# etcdをバックグラウンドで起動
etcd --name=$(hostname) \
--peer-key-file /etc/kubernetes/pki/api-server.key \
--peer-cert-file /etc/kubernetes/pki/api-server.crt \
--peer-trusted-ca-file /etc/kubernetes/pki/ca.crt \
--peer-client-cert-auth \
--key-file /etc/kubernetes/pki/api-server.key \
--cert-file /etc/kubernetes/pki/api-server.crt \
--trusted-ca-file /etc/kubernetes/pki/ca.crt \
--client-cert-auth \
--listen-peer-urls=https://0.0.0.0:2380 \
--listen-client-urls=https://0.0.0.0:2379 \
--advertise-client-urls https://$(hostname).internal:2379 \
--initial-advertise-peer-urls=https://$(hostname).internal:2380 \
--data-dir /var/lib/etcd \
--initial-cluster=control-0=https://control-0.internal:2380,control-1=https://control-1.internal:2380,control-2=https://control-2.internal:2380 \
--initial-cluster-state=new \
--initial-cluster-token=etcd-cluster-0 \
&
```

> TODO: Advertise は `127.0.0.1` のみとし、クライアントリスナは `127.0.0.1` のみBINDする

### etcdの動作確認

```shell-session:control-N
# データの書き込み
etcdctl put mykey "I am $(hostname)" \
--key /etc/kubernetes/pki/admin.key \
--cert /etc/kubernetes/pki/admin.crt \
--cacert /etc/kubernetes/pki/ca.crt \
--endpoints=https://$(hostname).internal:2379

# データの読み取り
etcdctl get mykey \
--key /etc/kubernetes/pki/admin.key \
--cert /etc/kubernetes/pki/admin.crt \
--cacert /etc/kubernetes/pki/ca.crt \
--endpoints=https://$(hostname).internal:2379

# クラスターメンバーの確認
etcdctl member list \
--key /etc/kubernetes/pki/admin.key \
--cert /etc/kubernetes/pki/admin.crt \
--cacert /etc/kubernetes/pki/ca.crt \
--endpoints=https://$(hostname).internal:2379

exit
```

### Kubernetesコンポーネントの起動

```shell-session:control-N
# kube-apiserverの起動
kube-apiserver \
--service-cluster-ip-range 192.168.64.0/24 \
--service-account-key-file /etc/kubernetes/pki/service-account.key \
--service-account-issuer /etc/kubernetes/pki/ca.crt \
--service-account-signing-key-file /etc/kubernetes/pki/service-account.key \
--etcd-servers https://control-0.internal:2379,https://control-1.internal:2379,https://control-2.internal:2379 \
--etcd-keyfile /etc/kubernetes/pki/api-server.key \
--etcd-certfile /etc/kubernetes/pki/api-server.crt \
--etcd-cafile /etc/kubernetes/pki/ca.crt \
--tls-cert-file=/etc/kubernetes/pki/api-server.crt \
--tls-private-key-file=/etc/kubernetes/pki/api-server.key \
--client-ca-file /etc/kubernetes/pki/ca.crt \
&

# kube-controller-managerのkubeconfig設定
(
  export KUBECONFIG=/etc/kubernetes/controller-manager.conf
  kubectl config set-cluster hardway \
    --certificate-authority=/etc/kubernetes/pki/ca.crt \
    --embed-certs=true \
    --server=https://$(hostname).internal:6443
  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=/etc/kubernetes/pki/kube-controller-manager.crt \
    --client-key=/etc/kubernetes/pki/kube-controller-manager.key \
    --embed-certs=true
  kubectl config set-context default \
    --cluster=hardway \
    --user=system:kube-controller-manager
  kubectl config use-context default
)

# kube-controller-managerの起動
kube-controller-manager \
--kubeconfig /etc/kubernetes/controller-manager.conf \
&

# kube-schedulerのkubeconfig設定
(
  export KUBECONFIG=/etc/kubernetes/scheduler.conf
  kubectl config set-cluster hardway \
    --certificate-authority=/etc/kubernetes/pki/ca.crt \
    --embed-certs=true \
    --server=https://$(hostname).internal:6443
  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=/etc/kubernetes/pki/kube-scheduler.crt \
    --client-key=/etc/kubernetes/pki/kube-scheduler.key \
    --embed-certs=true
  kubectl config set-context default \
    --cluster=hardway \
    --user=system:kube-scheduler
  kubectl config use-context default
)

# kube-schedulerの起動
kube-scheduler \
  --kubeconfig /etc/kubernetes/scheduler.conf \
  --leader-elect \
&
```

### kubectlの設定

`jumpbox`に戻り、`kubectl`でクラスタにアクセスするための設定を行います。

```shell-session:jumpbox
kubectl config set-cluster hardway \
  --server=https://control-0.internal:6443 \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true
kubectl config set-credentials admin \
  --client-certificate=/etc/kubernetes/pki/admin.crt \
  --client-key=/etc/kubernetes/pki/admin.key \
  --embed-certs=true
kubectl config set-context default \
  --cluster=hardway \
  --user=admin
kubectl config use-context default 

# コンポーネントの状態を確認
kubectl get componentstatuses
```

## 6. Workerノードの構築

各Workerノード (`worker-0`, `worker-1`, `worker-2`) に入り、コンテナランタイムと`kubelet`、`kube-proxy`を起動します。

### コンテナランタイムのセットアップ

参考資料:
- [コンテナランタイムを設定する](https://kubernetes.io/ja/docs/setup/production-environment/container-runtimes/)
- [CRI-O Packaging](https://github.com/cri-o/packaging/blob/main/README.md#distributions-using-deb-packages)

```shell-session:worker-N
# 必要なパッケージをインストール
apt-get update && apt-get install -y \
  conntrack \
  curl \
  ipset \
  socat \
  software-properties-common

# iptablesをlegacyモードに設定
apt install -y iptables
update-alternatives --set iptables /usr/sbin/iptables-legacy
update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
apt-cache policy nftables

# CRI-Oのインストール
# ref: https://github.com/cri-o/cri-o
curl https://storage.googleapis.com/cri-o/artifacts/cri-o.arm64.v1.33.3.tar.gz | tar xzf -
cd cri-o
./install

# CNIプラグインの設定
# ref: https://github.com/containernetworking/cni?tab=readme-ov-file#running-the-plugins
tee /etc/cni/net.d/192-mynet.conf <<EOF
{
	"cniVersion": "0.2.0",
	"name": "mynet",
	"type": "bridge",
	"bridge": "cni0",
	"isGateway": true,
	"ipMasq": true,
	"ipam": {
		"type": "host-local",
		"subnet": "192.168.64.128/25",
		"routes": [
			{ "dst": "0.0.0.0/0" }
		]
	}
}
EOF

tee /etc/cni/net.d/99-loopback.conf <<EOF
{
	"cniVersion": "0.2.0",
	"name": "lo",
	"type": "loopback"
}
EOF

# CRI-Oの起動
crio \
  --cgroup-manager cgroupfs\
&
```

### CRI-Oの単体テスト

参考: [crictl を使った Kubernetes ノードのデバッグ](https://qiita.com/cfg17771855/items/703ee2627cfafd276735)

```shell-session:worker-N
# Pod/コンテナ定義の作成
tee busybox-pod.json <<EOF
{
  "metadata": {
    "name": "busybox",
    "namespace": "default",
    "attempt": 1,
    "uid": "uuiduuiduuiduuiduuiduuid0"
  },
  "log_directory": "/tmp",
  "linux": {}
}
EOF

tee busybox-container.json <<EOF
{
  "metadata": {"name": "busybox"},
  "image":{"image": "busybox"},
  "command": ["top"],
  "log_path":"busybox.0.log",
  "linux": {}
}
EOF

# テスト実行
crictl pull busybox
crictl image
POD_ID=$(crictl runp busybox-pod.json)
echo ${POD_ID}
crictl pods --namespace default
CONTAINER_ID=$(crictl create ${POD_ID} busybox-container.json busybox-pod.json)
echo ${CONTAINER_ID}
crictl ps -a --no-trunc
crictl start ${CONTAINER_ID}
crictl exec -it ${CONTAINER_ID} sh
```

### Kubeletのセットアップ

```shell-session:worker-N
# kubeletのkubeconfig設定
(
  export KUBECONFIG=/etc/kubernetes/kubelet.conf
  kubectl config set-cluster hardway \
    --certificate-authority=/etc/kubernetes/pki/ca.crt \
    --embed-certs=true \
    --server=https://control-0.internal:6443
  kubectl config set-credentials system:node:$(hostname) \
    --client-certificate=/etc/kubernetes/pki/$(hostname).crt \
    --client-key=/etc/kubernetes/pki/$(hostname).key \
    --embed-certs=true
  kubectl config set-context default \
    --cluster=hardway \
    --user=system:node:$(hostname)
  kubectl config use-context default
)

# kubeletの設定ファイル作成
# ref: https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/
tee /etc/kubernetes/kubelet.yaml <<EOF
apiversion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/etc/kubernetes/pki/ca.crt"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "192.168.64.1"
podCIDR: "192.168.64.128/25"
runtimeRequestTimeout: "15m"
tlsCertFile: "/etc/kubernetes/pki/$(hostname).crt"
tlsPrivateKeyFile: "/etc/kubernetes/pki/$(hostname).key"
resolvConf: "/run/systemd/resolve/resolv.conf"
containerRuntimeEndpoint: /var/run/crio/crio.sock
EOF

# DNS設定
ln -s /run/systemd/resolve/{stub-,}resolv.conf

# kubeletの起動
kubelet \
  --config /etc/kubernetes/kubelet.yaml \
  --kubeconfig /etc/kubernetes/kubelet.conf \
&
```

### Kube-proxyのセットアップ

```shell-session:worker-N
# TODO: server with balancer

# kube-proxyのkubeconfig設定
(
  export KUBECONFIG=/etc/kubernetes/proxy.conf
  kubectl config set-cluster hardway \
    --certificate-authority=/etc/kubernetes/pki/ca.crt \
    --embed-certs=true \
    --server=https://control-0.internal:6443
  kubectl config set-credentials system:kube-proxy \
    --client-certificate=/etc/kubernetes/pki/kube-proxy.crt \
    --client-key=/etc/kubernetes/pki/kube-proxy.key \
    --embed-certs=true
  kubectl config set-context default \
    --cluster=hardway \
    --user=system:kube-proxy
  kubectl config use-context default
)

# kube-proxyの起動
kube-proxy \
  --cluster-cidr 192.168.64.0/24 \
  --kubeconfig /etc/kubernetes/proxy.conf \
  --proxy-mode iptables \
  --nodeport-addresses primary \
&
```

## 補足: Docker in container の動作確認

[公式ドキュメント](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)の手順に従い、Dockerをインストールして動作を確認します。

```shell-session
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
```

このままだと、以下のエラーで`dockerd`が起動に失敗します。
```
tail /var/log/docker.log
failed to start daemon: Error initializing network controller: error obtaining controller instance: failed to register "bridge" driver: failed to create NAT chain DOCKER: iptables failed: iptables --wait -t nat -N DOCKER: iptables v1.8.10 (nf_tables): Could not fetch rule set generation id: Invalid argument
 (exit status 4) 

root@control-1:/# iptables -t nat -N DOCKER
iptables v1.8.10 (nf_tables): Could not fetch rule set generation id: Invalid argument
```

[こちらの記事](https://marendasoft.eu/iptables-v1-8-4-nf_tables-could-not-fetch-rule-set-generation-id-invalid-argument/)を参考に、`iptables`をlegacyモードに切り替えます。

```shell-session
apt install iptables
update-alternatives --set iptables /usr/sbin/iptables-legacy
update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
apt-cache policy nftables

service docker start
docker ps
docker run hello-world
```

これで`docker`が動作するようになります。

##
