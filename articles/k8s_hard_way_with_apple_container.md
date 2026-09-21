---
title: "kubernetes_the_hard_way on apple/container"
emoji: "📦"
type: "tech"
topics: ["kubernetes", "kubernetes-the-hard-way", "apple/container" ]
published: false
---

### 概要

[kubernetes the hard way](https://github.com/kelseyhightower/kubernetes-the-hard-way) を [apple/container](https://github.com/apple/container) で試した記録です。

Component versions:

* [apple/container](https://github.com/apple/container/) v0.4.1
* [kubernetes](https://github.com/kubernetes/kubernetes) v1.34.0
* [cri-o](https://github.com/cri-o/cri-o) v1.34.0
* [cni](https://github.com/containernetworking/cni) v1.6.x #FIXME
* [etcd](https://github.com/etcd-io/etcd) v3.6.4


### 注記

- この記事は作成中です。
- ライセンスは [CC-BY-NC-SA-4.0](http://creativecommons.org/licenses/by-nc-sa/4.0/) とします。
- コマンド例のプロンプト `m4mac%` はmacOSのzsh、それ以外はコンテナ内の仮想マシンで実行することを示します。
- [Options for Highly Available Topology](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/) の Stacked etcd topology を採用します。Control Planeの各ノードにetcdを配置し、API-Serverは同じノードのetcdに接続する構成です。

### ファイル配置

TLS関連のファイルは、https://kubernetes.io/docs/setup/best-practices/certificates/ に従い、以下の場所に配置します。

- /etc/kubernetes/pki/etcd/ca.{key,crt}
- /etc/kubernetes/pki/apiserver-etcd-client.{key,crt}
- /etc/kubernetes/pki/ca.{key,crt}
- /etc/kubernetes/pki/apiserver.{key,crt}
- /etc/kubernetes/pki/apiserver-kubelet-client.{key,crt}
- /etc/kubernetes/pki/front-proxy-ca.{key,crt}
- /etc/kubernetes/pki/front-proxy-client.{key,crt}
- /etc/kubernetes/pki/etcd/server.{key,crt}
- /etc/kubernetes/pki/etcd/peer.{key,crt}
- /etc/kubernetes/pki/etcd/healthcheck-client.{key,crt}
- /etc/kubernetes/pki/sa.{key,pub}
- /etc/kubernetes/admin.conf
- /etc/kubernetes/super-admin.conf
- /etc/kubernetes/kubelet.conf
- /etc/kubernetes/controller-manager.conf
- /etc/kubernetes/scheduler.conf

## 1. 準備

### 前提条件

`apple/container` のネットワークを正しく設定するには、macOS 26 beta以降が必要です。以前のバージョンではコンテナ間通信ができません。

### Apple Container をインストールする

[apple/container](https://github.com/apple/container)の最新版（本記事執筆時点では0.4.1）を[ダウンロード](https://github.com/apple/container/releases/tag/0.4.1)し、インストールします。`brew install container` でも入るようです。

インストール後、`apple/container`を起動します。DNSとブリッジを確認しておきます。

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
m4mac% defaults read com.apple.container
{
    "dns.domain" = internal
}

m4mac ~ % ifconfig bridge100 inet
bridge100: flags=8a63<UP,BROADCAST,SMART,RUNNING,ALLMULTI,SIMPLEX,MULTICAST> mtu 1500
	options=63<RXCSUM,TXCSUM,TSO4,TSO6>
	inet 192.168.64.1 netmask 0xffffff00 broadcast 192.168.64.255

container create network az_a
matobaa@m4mac ~ % ifconfig bridge101 inet
bridge101: flags=8a63<UP,BROADCAST,SMART,RUNNING,ALLMULTI,SIMPLEX,MULTICAST> mtu 1500
	options=63<RXCSUM,TXCSUM,TSO4,TSO6>
	inet 192.168.65.1 netmask 0xffffff00 broadcast 192.168.65.255
```

## 2. 仮想マシンを作成する

まず、systemdが動くコンテナイメージを準備します:
```shell-session:m4mac
cat >Dockerfile <<-EOF
        FROM debian:13.0-slim as systemd
        RUN DEBIAN_FRONTEND=noninteractive apt-get update && apt-get install -y init systemd && rm -rf /var/lib/apt/lists/*
        CMD ["/sbin/init"]
EOF
container build -t systemd -f Dockerfile
```

そのイメージを使い、作業用1台、コントロールプレーン3台、ワーカー3台を起動します。

```shell-session:m4mac
container run -d --name jumpbox systemd
container run -d --name control-0 systemd
container run -d --name control-1 systemd
container run -d --name control-2 systemd
container run -d --name worker-0 systemd
container run -d --name worker-1 systemd
container run -d --name worker-2 systemd

container list | sort -k 5

: 実行結果の例'
ID         IMAGE                                               OS     ARCH   STATE    ADDR
buildkit   ghcr.io/apple/container-builder-shim/builder:0.6.0  linux  arm64  running  192.168.64.2
jumpbox    systemd:latest                                      linux  arm64  running  192.168.64.3
control-0  systemd:latest                                      linux  arm64  running  192.168.64.4
control-1  systemd:latest                                      linux  arm64  running  192.168.64.5
control-2  systemd:latest                                      linux  arm64  running  192.168.64.6
worker-0   systemd:latest                                      linux  arm64  running  192.168.64.7
worker-1   systemd:latest                                      linux  arm64  running  192.168.64.8
worker-2   systemd:latest                                      linux  arm64  running  192.168.64.9
:'
```

## 3. バイナリとTLS鍵・証明書を準備する

作業用の`jumpbox`コンテナに入り、各ノードで必要となるバイナリと鍵と証明書を準備します。

```shell-session:m4mac
: jumpboxに入る:
container exec -it jumpbox bash
```

```shell-session:jumpbox
# install required packages
apt-get update && apt-get -y install wget curl openssl dnsutils

# prepare target directories
cd
mkdir -p /root/control/usr/local/bin
mkdir -p /root/worker/usr/local/bin

# download and extract the etcd binary
curl https://storage.googleapis.com/etcd/v3.6.4/etcd-v3.6.4-linux-arm64.tar.gz | tar xzf -
install etcd-v3.6.4-linux-arm64/etcd* /root/control/usr/local/bin/

# download and extract the kubernetes binaries
# ref: https://kubernetes.io/releases/download/#binaries
wget -q --show-progress \
https://dl.k8s.io/v1.34.0/bin/linux/arm64/kube-apiserver \
https://dl.k8s.io/v1.34.0/bin/linux/arm64/kube-controller-manager \
https://dl.k8s.io/v1.34.0/bin/linux/arm64/kube-scheduler \
https://dl.k8s.io/v1.34.0/bin/linux/arm64/kubectl \
https://dl.k8s.io/v1.34.0/bin/linux/arm64/kube-proxy \
https://dl.k8s.io/v1.34.0/bin/linux/arm64/kubelet
install -m 755 kubectl kube-apiserver kube-controller-manager kube-scheduler kubelet /root/control/usr/local/bin/
install -m 755 kubectl kube-proxy kubelet /root/worker/usr/local/bin/

# prepare PKI target directries
mkdir -p /etc/kubernetes/pki/etcd
cd /etc/kubernetes/pki/

# ETCD CA; selfsign > etcd/ca.{key,csr,crt}
openssl ecparam -name prime256v1 -genkey -noout -out etcd/ca.key
openssl req -new -key etcd/ca.key -subj "/CN=ETCD-CA" -out etcd/ca.csr
openssl x509 -req -in etcd/ca.csr -signkey etcd/ca.key -days 365 -out etcd/ca.crt -extfile - <<-EOF
  basicConstraints=CA:TRUE
  keyUsage=keyCertSign,cRLSign
EOF

# ETCD Peer; signed by etcd-ca -> etcd/peer.{key,csr,crt}
openssl ecparam -name prime256v1 -genkey -noout -out etcd/peer.key
openssl req -new -key etcd/peer.key -subj "/CN=kube-etcd-peer" -out etcd/peer.csr
openssl x509 -req -in etcd/peer.csr -CA etcd/ca.crt -CAkey etcd/ca.key -days 365 -out etcd/peer.crt -extfile - <<-EOF
  basicConstraints=CA:FALSE
  keyUsage=digitalSignature,keyEncipherment,keyAgreement
  extendedKeyUsage=serverAuth,clientAuth
  subjectAltName=DNS:control-0.internal,DNS:control-1.internal,DNS:control-2.internal
EOF

# ETCD Server; signed by etcd-ca -> etcd/server.{key,csr,crt}
openssl ecparam -name prime256v1 -genkey -noout -out etcd/server.key
openssl req -new -key etcd/server.key -subj "/CN=kube-etcd" -out etcd/server.csr
openssl x509 -req -in etcd/server.csr -CA etcd/ca.crt -CAkey etcd/ca.key -days 365 -out etcd/server.crt -extfile - <<-EOF
  basicConstraints=CA:FALSE
  keyUsage=digitalSignature,keyEncipherment,keyAgreement
  extendedKeyUsage=serverAuth,clientAuth
  subjectAltName=DNS:control-0.internal,DNS:control-1.internal,DNS:control-2.internal,IP:$(dig +short control-0.internal),IP:$(dig +short control-1.internal),IP:$(dig +short control-2.internal)
EOF

# Kubernetes API Server as ETCD Client; signed by etcd-ca -> apiserver-etcd-client.{key,csr,crt}
openssl ecparam -name prime256v1 -genkey -noout -out apiserver-etcd-client.key
openssl req -new -key apiserver-etcd-client.key -subj "/CN=kube-apiserver-etcd-client" -out apiserver-etcd-client.csr
openssl x509 -req -in apiserver-etcd-client.csr -CA etcd/ca.crt -CAkey etcd/ca.key -days 365 -out apiserver-etcd-client.crt -extfile - <<-EOF
  basicConstraints=CA:FALSE
  keyUsage=digitalSignature,keyEncipherment,keyAgreement
  extendedKeyUsage=clientAuth
EOF

# Kubernetes CA; selfsigned -> ca.{key,csr,crt}
openssl ecparam -name prime256v1 -genkey -noout -out ca.key
openssl req -new -key ca.key -subj "/CN=kubernetes-ca" -out ca.csr
openssl x509 -req -in ca.csr -signkey ca.key -days 365 -out ca.crt -extfile - <<-EOF
  basicConstraints=CA:TRUE
  keyUsage=keyCertSign,cRLSign
EOF

# Kubernetes API Server; signed by Kubernetes CA -> apiserver.{key,csr,crt}
openssl ecparam -name prime256v1 -genkey -noout -out apiserver.key
openssl req -new -key apiserver.key -subj "/CN=kube-apiserver" -out apiserver.csr
openssl x509 -req -in apiserver.csr -CA ca.crt -CAkey ca.key -days 365 -out apiserver.crt -extfile - <<-EOF
  basicConstraints=CA:FALSE
  keyUsage=digitalSignature,keyEncipherment,keyAgreement
  extendedKeyUsage=serverAuth
  subjectAltName=DNS:control-0.internal,DNS:control-1.internal,DNS:control-2.internal
EOF

# admin; signed by Kubernetes CA -> admin.{key,csr,crt}
openssl ecparam -name prime256v1 -genkey -noout -out admin.key
openssl req -new -key admin.key -subj "/CN=admin/O=system:masters" -out admin.csr
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -days 365 -out admin.crt -extfile - <<-EOF
  basicConstraints=CA:FALSE
  extendedKeyUsage=clientAuth
  keyUsage=digitalSignature,keyEncipherment,keyAgreement
EOF

# kube-controller-manager; signed by Kubernetes CA -> kube-controller-manager.{key,csr,crt}
openssl ecparam -name prime256v1 -genkey -noout -out kube-controller-manager.key
openssl req -new -key kube-controller-manager.key -subj "/CN=system:kube-controller-manager" -out kube-controller-manager.csr
openssl x509 -req -in kube-controller-manager.csr -CA ca.crt -CAkey ca.key -days 365 -out kube-controller-manager.crt -extfile - <<-EOF
  basicConstraints=CA:FALSE
  extendedKeyUsage=clientAuth
  keyUsage=digitalSignature,keyEncipherment,keyAgreement
EOF



# 不要になったCSR(証明書署名要求)ファイルを削除
rm *.csr etcd/*.csr

exit
```

- admin.crt は O=system:masters であること ∵ ClusterRoleBinding/cluster-admin
- kube-controller-manager.crt は CN=system:kube-controller-manager であること ∵ ClusterRoleBinding/system:kube-controller-manager


> **注:** v1.33.0では、`ed25519` 鍵を使用すると、API-Server起動時に `unknown private key type ed25519.PrivateKey` エラーが発生しました。

## 4. 各ノードへファイルを配布する

`jumpbox`で準備した鍵とバイナリを、各Control PlaneノードとWorkerノードにコピーします。

```shell-session:m4mac
: Copy files to Controlplane nodes
container exec jumpbox tar cf - etc/kubernetes/pki -C /root/control usr/local/bin \
   | tee >(container exec -i control-0 tar xf -) \
   | tee >(container exec -i control-1 tar xf -) \
   |       container exec -i control-2 tar xf -

: Copy files to Worker nodes
container exec jumpbox tar cf - etc/kubernetes/pki -C /root/worker usr/local/bin \
   | tee >(container exec -i worker-0 tar xf -) \
   | tee >(container exec -i worker-1 tar xf -) \
   |       container exec -i worker-2 tar xf -
```

## 5. Control Planeノードを構築する

各Control Planeノード (`control-0`, `control-1`, `control-2`) に入り、etcdとKubernetesコンポーネントを起動します。

```
container exec -it control-0 bash
: control-1, control-2 も同様に
```

### etcdを構成する

```shell-session:control-N
HOSTNAME=$(hostname)
ETCD0=${HOSTNAME/-[0-9]/-0}
ETCD1=${HOSTNAME/-[0-9]/-1}
ETCD2=${HOSTNAME/-[0-9]/-2}

# Create the etcd.service systemd unit file:
cat <<EOF >/etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name=${HOSTNAME} \\
  \\
  --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt \\
  --peer-key-file=/etc/kubernetes/pki/etcd/peer.key \\
  --peer-client-cert-auth \\
  --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt \\
  --listen-peer-urls=https://$(hostname -i):2380 \\
  --initial-advertise-peer-urls=https://${HOSTNAME}.internal:2380 \\
  \\
  --cert-file=/etc/kubernetes/pki/etcd/server.crt \\
  --key-file=/etc/kubernetes/pki/etcd/server.key \\
  --client-cert-auth \\
  --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt \\
  --listen-client-urls=https://$(hostname -i):2379,https://127.0.0.1:2379 \\
  --advertise-client-urls=https://${HOSTNAME}.internal:2379 \\
  \\
  --initial-cluster-token=etcd-cluster-0 \\
  --initial-cluster=${ETCD0}=https://${ETCD0}.internal:2380,${ETCD1}=https://${ETCD1}.internal:2380,${ETCD2}=https://${ETCD2}.internal:2380 \\
  \\
  --initial-cluster-state=new \\
  --data-dir=/var/lib/etcd

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# start etcd
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
```

> TODO: Advertise は `127.0.0.1` のみとし、クライアントリスナは `127.0.0.1` のみBINDする

> ここまでを3台で実行する

### etcdの動作を確認する

```shell-session:control-N
export ETCDCTL_KEY=/etc/kubernetes/pki/apiserver-etcd-client.key
export ETCDCTL_CERT=/etc/kubernetes/pki/apiserver-etcd-client.crt
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_ENDPOINTS=https://control-0.internal:2379

# データを書き込む
etcdctl put mykey "I am $(hostname)" --endpoints=https://control-0.internal:2379

# データを読み取る
etcdctl get mykey --endpoints=https://control-1.internal:2379

# クラスターメンバーを確認する
etcdctl member list --endpoints=https://control-2.internal:2379

```

### kube-apiserver を構成する

```shell-session:control-N
HOSTNAME=$(hostname)
ETCD0=${HOSTNAME/-[0-9]/-0}
ETCD1=${HOSTNAME/-[0-9]/-1}
ETCD2=${HOSTNAME/-[0-9]/-2}

# Create the kube-apiserver.service systemd unit file:
cat <<EOF >/etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --etcd-servers=https://${ETCD0}.internal:2379,https://${ETCD1}.internal:2379,https://${ETCD2}.internal:2379 \\
  --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt \\
  --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key \\
  --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt \\
  \\
  --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt \\
  --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key \\
  --kubelet-certificate-authority=/etc/kubernetes/pki/ca.crt \\
  \\
  --bind-address=0.0.0.0 \\
  --tls-cert-file=/etc/kubernetes/pki/apiserver.crt \\
  --tls-private-key-file=/etc/kubernetes/pki/apiserver.key \\
  --client-ca-file=/etc/kubernetes/pki/ca.crt \\
  \\
  --service-account-issuer=/etc/kubernetes/pki/ca.crt \\
  --service-account-key-file=/etc/kubernetes/pki/sa.key \\
  --service-account-signing-key-file=/etc/kubernetes/pki/sa.key \\
  \\
  --service-cluster-ip-range=172.18.0.0/24 \\
  --external-hostname=${HOSTNAME}.internal \\
  --authorization-mode=Node,RBAC \\

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### kube-controller-manager を構成する

```
# Generate a kubeconfig file for the kube-controller-manager service:
(
  cd /etc/kubernetes
  export KUBECONFIG=controller-manager.conf
  kubectl config set-cluster hardway \
    --certificate-authority=pki/ca.crt \
    --server=https://$(hostname).internal:6443
  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=pki/kube-controller-manager.crt \
    --client-key=pki/kube-controller-manager.key
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

## 6. Workerノードを構築する

各Workerノード (`worker-0`, `worker-1`, `worker-2`) に入り、コンテナランタイムと`kubelet`、`kube-proxy`を起動します。

### コンテナランタイムをセットアップする

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
curl https://storage.googleapis.com/cri-o/artifacts/cri-o.arm64.v1.34.0.tar.gz | tar xzf -
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
<<EOF cat > busybox-pod.json
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

<<EOF cat > busybox-container.json
{
  "metadata": {"name": "busybox"},
  "image": {"image": "busybox"},
  "command": ["top"],
  "log_path": "busybox.0.log",
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
/usr/local/bin/kubelet -v 15 --container-runtime-endpoint unix:///var/run/crio/crio.sock



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
