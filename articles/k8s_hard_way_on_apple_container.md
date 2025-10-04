---
title: "kubernetes_the_hard_way on apple/container"
emoji: "📦"
type: "tech"
topics: ["kubernetes", "applecontainer" ]
published: false
---

### 概要

[kubernetes the hard way](https://github.com/kelseyhightower/kubernetes-the-hard-way) を [apple/container](https://github.com/apple/container) で試した記録です。

Component versions:

* [apple/container](https://github.com/apple/container/) v0.4.1
* [kubernetes](https://github.com/kubernetes/kubernetes) v1.33.0
* [cri-o](https://github.com/cri-o/cri-o) v1.33.4
* [cni](https://github.com/containernetworking/cni) v1.6.x #FIXME
* [etcd](https://github.com/etcd-io/etcd) v3.6.4


### 注記

- この記事は作成中です。
- ライセンスは [CC-BY-NC-SA-4.0](http://creativecommons.org/licenses/by-nc-sa/4.0/) とします。
- コマンド例のタブに`m4mac` があるものは母艦側macOSのターミナル(zsh)で、それ以外はapple/containerの仮想マシン内で実行することを示します。
- [Options for Highly Available Topology](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/) の Stacked etcd topology を採用します。つまり、Control Planeの各ノードにetcdを配置し、API-Serverは同じノードのetcdに接続する構成です。

## 1. 準備

### 前提条件

`apple/container` のネットワークを正しく設定するには、macOS 26 beta以降が必要です。以前のバージョンではコンテナ間通信ができません。

### Apple Container をインストールする

[apple/container](https://github.com/apple/container)の最新版（本記事執筆時点では0.4.1）を[ダウンロード](https://github.com/apple/container/releases/tag/0.4.1)し、インストールします。`brew install container` でも入るようです。

インストール後、`apple/container`を起動します。DNSとブリッジを確認しておきます。

```shell:m4mac
setopt interactivecomments
# ↑ これを実行することで、そのzshでは#が行コメント扱いになります
# 以下、出力例を行コメント等で示します

container --version
# container CLI version 0.4.1 (build: release, commit: 4ac18b5)

container system start
# Verifying apiserver is running...
# No default kernel configured.
# Install the recommended default kernel from [https://github.com/kata-containers/kata-containers/releases/download/3.17.0/kata-static-3.17.0-arm64.tar.xz]? [Y/n]: Y
# Installing kernel...

container system status
# apiserver is running
# application data root: /Users/matobaa/Library/Application Support/com.apple.container/
# application install root: /usr/local/
# container-apiserver version: container-apiserver version 0.4.1 (build: release, commit: 4ac18b5)
# container-apiserver commit: 4ac18b54f376b03ce24448abdd86a4a3371bad43

## container system dns create internal
## container system dns default set internal
defaults read com.apple.container
#{
#    "dns.domain" = internal
#}
ifconfig -l
# lo0 gif0 stf0 anpi0 anpi1 anpi3 en0 en5 en6 en7 en2 en3 en4 bridge0 ap1 en1 awdl0 llw0 utun0 utun1 utun2 utun3
```

## 2. 仮想マシンを作成する

まず、systemdが動くコンテナイメージを準備します:
```shell:m4mac
cat >Dockerfile <<-EOF
  FROM debian:13.0-slim as systemd
  RUN DEBIAN_FRONTEND=noninteractive apt-get update \
      && apt-get install -y init systemd vim procps \
      && rm -rf /var/lib/apt/lists/*
  CMD ["/sbin/init"]
EOF
container build -t systemd -f Dockerfile
```

作成したイメージを使い、作業用1台、コントロールプレーン3台、ワーカー3台を起動します。

```shell:m4mac
container run -d --name jumpbox systemd
container run -d --name control-0 systemd
container run -d --name control-1 systemd
container run -d --name control-2 systemd
container run -d --name worker-0 systemd
container run -d --name worker-1 systemd
container run -d --name worker-2 systemd

container list | sort -k 5
: 出力例<<_____
ID         IMAGE           OS     ARCH   STATE    ADDR
jumpbox    systemd:latest  linux  arm64  running  192.168.64.3
control-0  systemd:latest  linux  arm64  running  192.168.64.4
control-1  systemd:latest  linux  arm64  running  192.168.64.5
control-2  systemd:latest  linux  arm64  running  192.168.64.6
worker-0   systemd:latest  linux  arm64  running  192.168.64.7
worker-1   systemd:latest  linux  arm64  running  192.168.64.8
worker-2   systemd:latest  linux  arm64  running  192.168.64.9
_____
ifconfig -l
# lo0 gif0 stf0 anpi0 anpi1 anpi3 en0 en5 en6 en7 en2 en3 en4 bridge0 ap1 en1 awdl0 llw0 utun0 utun1 utun2 utun3 vmenet0 bridge100 vmenet1 vmenet2 vmenet3 vmenet4 vmenet5 vmenet6
ifconfig bridge100 inet
# bridge100: flags=8a63<UP,BROADCAST,SMART,RUNNING,ALLMULTI,SIMPLEX,MULTICAST> mtu 1500
#   options=63<RXCSUM,TXCSUM,TSO4,TSO6>
#   inet 192.168.64.1 netmask 0xffffff00 broadcast 192.168.64.255
### containerインスタンスのIPアドレス範囲は 192.168.64.xx/24 になるようだ ###
### containerインスタンス1つにつきvmenetが1つ割当たっているようだ ###
```

## 3. バイナリとTLS鍵・証明書を準備する

[ベストプラクティス](https://kubernetes.io/docs/setup/best-practices/certificates/)に従い、TLS関連のファイルを以下の場所に作成していきます:
```
: TLS Files <<_____
/etc/kubernetes/pki/etcd/ca.{key,crt}
/etc/kubernetes/pki/apiserver-etcd-client.{key,crt}
/etc/kubernetes/pki/ca.{key,crt}
/etc/kubernetes/pki/apiserver.{key,crt}
/etc/kubernetes/pki/apiserver-kubelet-client.{key,crt}
/etc/kubernetes/pki/front-proxy-ca.{key,crt}
/etc/kubernetes/pki/front-proxy-client.{key,crt}
/etc/kubernetes/pki/etcd/server.{key,crt}
/etc/kubernetes/pki/etcd/peer.{key,crt}
/etc/kubernetes/pki/etcd/healthcheck-client.{key,crt}
/etc/kubernetes/pki/sa.{key,pub}
_____
: Configs <<_____
/etc/kubernetes/admin.conf
/etc/kubernetes/super-admin.conf
/etc/kubernetes/kubelet.conf
/etc/kubernetes/controller-manager.conf
/etc/kubernetes/scheduler.conf
~/.kube/config
_____
```

作業用の`jumpbox`コンテナに入り、各ノードで必要となるバイナリと鍵と証明書を準備します。

```shell:m4mac
# jumpboxに入る
container exec -it jumpbox bash
```

```shell:jumpbox
# install required packages
apt-get update && apt-get -y install wget curl openssl dnsutils

# prepare target directories
cd
mkdir -p /root/control/usr/local/bin
mkdir -p /root/worker/usr/local/bin

# download and extract the etcd binary
curl https://storage.googleapis.com/etcd/v3.6.4/etcd-v3.6.4-linux-arm64.tar.gz | tar xzf -
install -m 755 etcd-v3.6.4-linux-arm64/etcd* /root/control/usr/local/bin/
install -m 755 etcd-v3.6.4-linux-arm64/etcd* /usr/local/bin/

# download and extract the kubernetes binaries
# ref: https://kubernetes.io/releases/download/#binaries

wget -q --show-progress \
  https://dl.k8s.io/v1.33.0/bin/linux/arm64/kube-apiserver \
  https://dl.k8s.io/v1.33.0/bin/linux/arm64/kube-controller-manager \
  https://dl.k8s.io/v1.33.0/bin/linux/arm64/kube-scheduler \
  https://dl.k8s.io/v1.33.0/bin/linux/arm64/kubectl \
  https://dl.k8s.io/v1.33.0/bin/linux/arm64/kube-proxy \
  https://dl.k8s.io/v1.33.0/bin/linux/arm64/kubelet
install -m 755 kubectl kube-apiserver kube-controller-manager kube-scheduler kubelet /root/control/usr/local/bin/
install -m 755 kubectl kube-proxy kubelet /root/worker/usr/local/bin/
install -m 755 kubectl /usr/local/bin/

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
# The subject must have O=system:masters cf. ClusterRoleBinding/cluster-admin
openssl ecparam -name prime256v1 -genkey -noout -out admin.key
openssl req -new -key admin.key -subj "/CN=admin/O=system:masters" -out admin.csr
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -days 365 -out admin.crt -extfile - <<-EOF
  basicConstraints=CA:FALSE
  extendedKeyUsage=clientAuth
  keyUsage=digitalSignature,keyEncipherment,keyAgreement
EOF

# kube-controller-manager; signed by Kubernetes CA -> kube-controller-manager.{key,csr,crt}
# The subject must be CN=system:kube-controller-manager cf. ClusterRoleBinding/system:kube-controller-manager
openssl ecparam -name prime256v1 -genkey -noout -out kube-controller-manager.key
openssl req -new -key kube-controller-manager.key -subj "/CN=system:kube-controller-manager" -out kube-controller-manager.csr
openssl x509 -req -in kube-controller-manager.csr -CA ca.crt -CAkey ca.key -days 365 -out kube-controller-manager.crt -extfile - <<-EOF
  basicConstraints=CA:FALSE
  extendedKeyUsage=clientAuth
  keyUsage=digitalSignature,keyEncipherment,keyAgreement
EOF

# kube-scheduler; signed by Kubernetes CA -> kube-scheduler.{key,csr,crt}
# The subject must be CN=system:kube-scheduler cf. ClusterRoleBinding/system:kube-scheduler
openssl ecparam -name prime256v1 -genkey -noout -out kube-scheduler.key
openssl req -new -key kube-scheduler.key -subj "/CN=system:kube-scheduler" -out kube-scheduler.csr
openssl x509 -req -in kube-scheduler.csr -CA ca.crt -CAkey ca.key -days 365 -out kube-scheduler.crt -extfile - <<-EOF
  basicConstraints=CA:FALSE
  extendedKeyUsage=clientAuth
  keyUsage=digitalSignature,keyEncipherment,keyAgreement
EOF

# kube-proxy; signed by Kubernetes CA -> kube-proxy.{key,csr,crt}
# The subject must be CN=system:kube-proxy cf. ClusterRoleBinding/system:kube-proxy
openssl ecparam -name prime256v1 -genkey -noout -out kube-proxy.key
openssl req -new -key kube-proxy.key -subj "/CN=system:kube-proxy" -out kube-proxy.csr
openssl x509 -req -in kube-proxy.csr -CA ca.crt -CAkey ca.key -days 365 -out kube-proxy.crt -extfile - <<-EOF
  basicConstraints=CA:FALSE
  extendedKeyUsage=clientAuth
  keyUsage=digitalSignature,keyEncipherment,keyAgreement
EOF

# kubeletの鍵と証明書を作成、Kubernetes CAで署名
# The subject must be O=system:nodes/CN=system:node:$(hostname) cf. ClusterRoleBinding/system:node
for i in worker-{0,1,2}; do
  openssl ecparam -name prime256v1 -genkey -noout -out ${i}.key
  openssl req -new -key ${i}.key -subj "/CN=system:node:${i}/O=system:nodes" -out ${i}.csr
  openssl x509 -req -in ${i}.csr -CA ca.crt -CAkey ca.key -days 365 -out ${i}.crt -extfile - <<-EOF
    basicConstraints=CA:FALSE
    keyUsage=digitalSignature,keyEncipherment,keyAgreement
    extendedKeyUsage=serverAuth,clientAuth
    subjectAltName=DNS:${i}
EOF
done

# 各コンポーネントの鍵と証明書を作成、Kubernetes CAで署名
# for i in  kube-proxy ; do
for i in sa apiserver-kubelet-client; do
  openssl ecparam -name prime256v1 -genkey -noout -out ${i}.key
  openssl req -new -key ${i}.key -subj /CN=${i} -out ${i}.csr
  openssl x509 -req -in ${i}.csr -CA ca.crt -CAkey ca.key -days 365 -out ${i}.crt -extfile - <<-EOF
    basicConstraints=CA:FALSE
    keyUsage=digitalSignature,keyEncipherment,keyAgreement
    extendedKeyUsage=clientAuth
EOF
done

# 不要になったCSR(証明書署名要求)ファイルを削除
rm *.csr etcd/*.csr

exit
```
> **注:** v1.33.0では、`ed25519` 鍵を使用すると、API-Server起動時に `unknown private key type ed25519.PrivateKey` エラーが発生しました。

## 4. 各ノードへファイルを配布する

`jumpbox`で準備した鍵とバイナリを、各Control PlaneノードとWorkerノードにコピーします。

```shell:m4mac
# Copy files to Controlplane nodes
container exec jumpbox tar cf - etc/kubernetes/pki -C /root/control usr/local/bin \
   | tee >(container exec -i control-0 tar xf -) \
   | tee >(container exec -i control-1 tar xf -) \
   |       container exec -i control-2 tar xf -

# Copy files to Worker nodes
container exec jumpbox tar cf - etc/kubernetes/pki -C /root/worker usr/local/bin \
   | tee >(container exec -i worker-0 tar xf -) \
   | tee >(container exec -i worker-1 tar xf -) \
   |       container exec -i worker-2 tar xf -
```

## 5. Control Planeノードを構築する

各Control Planeノード (`control-0`, `control-1`, `control-2`) に入り、etcdとKubernetesコンポーネントを起動します。

```shell:m4mac
container exec -it control-0 bash
# control-1, control-2 も同様に
```

### etcdを構成する

```shell:control-N
HOSTNAME=$(hostname)
ETCD0=${HOSTNAME/-[0-9]/-0}
ETCD1=${HOSTNAME/-[0-9]/-1}
ETCD2=${HOSTNAME/-[0-9]/-2}

# Create the etcd.service systemd unit file:
<<EOF sed -e "/^ *#/d" >/etc/systemd/system/etcd.service
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
  --listen-peer-urls=https://0.0.0.0:2380 \\
  --initial-advertise-peer-urls=https://${HOSTNAME}.internal:2380 \\
  \\
  --cert-file=/etc/kubernetes/pki/etcd/server.crt \\
  --key-file=/etc/kubernetes/pki/etcd/server.key \\
  --client-cert-auth \\
  --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt \\
  --listen-client-urls=https://0.0.0.0:2379 \\
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

: ここまでを3台で実行する

### etcdの動作を確認する

```shell:control-N
export ETCDCTL_KEY=/etc/kubernetes/pki/apiserver-etcd-client.key
export ETCDCTL_CERT=/etc/kubernetes/pki/apiserver-etcd-client.crt
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt

# put a data
etcdctl put mykey "I'm $(hostname)" --endpoints=https://control-0.internal:2379

# get the data
etcdctl get mykey --endpoints=https://control-1.internal:2379

# info
etcdctl member list --endpoints=https://control-2.internal:2379
```

### : kube-apiserver を構成する

```shell:control-N
HOSTNAME=$(hostname)
ETCD0=${HOSTNAME/-[0-9]/-0}
ETCD1=${HOSTNAME/-[0-9]/-1}
ETCD2=${HOSTNAME/-[0-9]/-2}

# Create the kube-apiserver.service systemd unit file:
<<EOF sed -e "/^ *#/d" >/etc/systemd/system/kube-apiserver.service
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
  --service-account-issuer=https://kubernetes.default.svc \\
  --service-account-key-file=/etc/kubernetes/pki/sa.crt \\
  --service-account-signing-key-file=/etc/kubernetes/pki/sa.key \\
  \\
  --service-cluster-ip-range=172.20.0.0/16 \\
  --external-hostname=${HOSTNAME}.internal \\
  --authorization-mode=Node,RBAC \\
  # https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/
  # --encryption-provider-config FIXME

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### : kube-controller-manager を構成する

```shell:control-N
# Generate a kubeconfig file for the kube-controller-manager service:
(
  cd /etc/kubernetes
  export KUBECONFIG=controller-manager.conf
  kubectl config set-cluster thecluster --certificate-authority=pki/ca.crt --server=https://$(hostname).internal:6443
  kubectl config set-credentials theuser --client-certificate=pki/kube-controller-manager.crt --client-key=pki/kube-controller-manager.key
  kubectl config set-context default --cluster=thecluster --user=theuser
  kubectl config use-context default
)

# Create the kube-controller-manager.service systemd unit file:
<<EOF sed -e "/^ *#/d" >/etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --kubeconfig=/etc/kubernetes/controller-manager.conf \\
  --use-service-account-credentials \\
  --allocate-node-cidrs \\
  --cluster-cidr=172.17.0.0/16 \\
  --service-cluster-ip-range=172.20.0.0/16 \\
  --leader-elect
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```
: escaped<<_____
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
_____
```

### : kube-scheduler を構成する

```shell:control-N
# Generate a kubeconfig file for the kube-scheduler service:
(
  cd /etc/kubernetes
  export KUBECONFIG=scheduler.conf
  kubectl config set-cluster thecluster --certificate-authority=pki/ca.crt --server=https://$(hostname).internal:6443
  kubectl config set-credentials theuser --client-certificate=pki/kube-scheduler.crt --client-key=pki/kube-scheduler.key
  kubectl config set-context default --cluster=thecluster --user=theuser
  kubectl config use-context default
)

# Create the kube-scheduler.service systemd unit file:
<<EOF sed -e "/^ *#/d" >/etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --kubeconfig=/etc/kubernetes/scheduler.conf \\
  --leader-elect
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# Start the Controller Services
systemctl daemon-reload
systemctl enable kube-apiserver kube-controller-manager kube-scheduler
systemctl start kube-apiserver; sleep 5
systemctl start kube-controller-manager; sleep 5
systemctl start kube-scheduler
```

### Verification

`jumpbox`に戻り、`kubectl`でクラスタにアクセスするための設定を行います。

```shell:jumpbox
# Generate a kubeconfig file for the kube-scheduler service:
(
  # ホスト名、ただしjumpboxの場合はcontrol-0とする、worker-Nの場合はcontrol-Nとする
  HOSTNAME=$(hostname)
  HOSTNAME=${HOSTNAME/jumpbox/control-0}
  HOSTNAME=${HOSTNAME/worker/control}

  cd /etc/kubernetes
  unset KUBECONFIG
  kubectl config set-cluster thecluster --certificate-authority=pki/ca.crt --server=https://${HOSTNAME}.internal:6443
  kubectl config set-credentials theuser --client-certificate=pki/admin.crt --client-key=pki/admin.key
  kubectl config set-context default --cluster=thecluster --user=theuser
  kubectl config use-context default
)

# コンポーネントの状態を確認
kubectl get componentstatuses
: 出力例<<_____
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE   ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   ok
_____
kubectl cluster-info
: 出力例<<_____
Kubernetes control plane is running at https://control-0.internal:6443
_____


# RBACを追加する。kube-apiserver -> kubelet のTLS証明書により user=apiserver-kubelet-client を使う
<<EOF kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: apiserver-kubelet-client
EOF

kubectl get clusterroles system:kube-apiserver-to-kubelet
# NAME                               CREATED AT
# system:kube-apiserver-to-kubelet   2025-09-22T11:02:57Z
kubectl get clusterrolebindings system:kube-apiserver
#NAME                    #ROLE                                           AGE
#system:kube-apiserver   ClusterRole/#system:kube-apiserver-to-kubelet   2m4s
```

## 6. Workerノードを構築する

各Workerノード (`worker-0`, `worker-1`, `worker-2`) に入り、コンテナランタイムと`kubelet`、`kube-proxy`を起動します。

```shell:m4mac
# jumpboxに入る
container exec -it worker-0 bash
# worker-1, worker-2 も同様に
```

### コンテナランタイムをセットアップする

参考資料:
- [コンテナランタイムを設定する](https://kubernetes.io/ja/docs/setup/production-environment/container-runtimes/)
- [CRI-O Packaging](https://github.com/cri-o/packaging/blob/main/README.md#distributions-using-deb-packages)
- https://github.com/cri-o/cri-o/blob/main/install.md

```shell:worker-N
# 必要なパッケージをインストール
apt-get update && apt-get install -y conntrack curl ipset socat iptables watchdog

# iptablesをlegacyモードに設定
update-alternatives --set iptables /usr/sbin/iptables-legacy
update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
apt-cache policy nftables

# CRI-Oのインストール
# ref: https://github.com/cri-o/cri-o
curl https://storage.googleapis.com/cri-o/artifacts/cri-o.arm64.v1.33.4.tar.gz | tar xzf -
cd cri-o; ./install

# CNIプラグインの設定
# ref: https://github.com/cri-o/packaging/blob/main/README.md#configure-a-container-network-interface-cni-plugin
# ref: https://github.com/containernetworking/cni?tab=readme-ov-file#running-the-plugins
OCTET=$(hostname | cut -d- -f 2)
tee /etc/cni/net.d/172-mynet.conf <<EOF
{
	"cniVersion": "0.2.0",
	"name": "mynet",
	"type": "bridge",
	"bridge": "cni0",
	"isGateway": true,
	"ipMasq": true,
	"ipam": {
		"type": "host-local",
		"subnet": "172.17.${OCTET}.0/24",
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

# Create the crio.service systemd unit file:
<<EOF sed -e "/^ *#/d" >/etc/systemd/system/crio.service
[Unit]
Description=Lightweight Container Runtime for Kubernetes
Documentation=https://cri-o.io/

[Service]
ExecStart=/usr/local/bin/crio \\
  --cgroup-manager cgroupfs
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# CRI-Oの起動
# crio --cgroup-manager cgroupfs &
systemctl daemon-reload
systemctl enable crio
systemctl start crio
```

### CRI-Oの単体テスト

参考:
- [crictl を使った Kubernetes ノードのデバッグ](https://qiita.com/cfg17771855/items/703ee2627cfafd276735)
- https://github.com/cri-o/cri-o/blob/main/tutorials/crictl.md

```shell:worker-N
# verification
crictl --runtime-endpoint unix:///var/run/crio/crio.sock version
: 出力例<<_____
Version:  0.1.0
RuntimeName:  cri-o
RuntimeVersion:  1.34.0
RuntimeApiVersion:  v1
_____

# define the pod/containers for test
<<EOF cat > pod.json
{
  "metadata": {
    "name": "default",
    "namespace": "default",
    "attempt": 1,
    "uid": "uuiduuiduuiduuiduuiduuid0"
  },
  "log_directory": "/tmp",
  "linux": {}
}
EOF

<<EOF cat > nginx.json
{
  "metadata": {"name": "nginx"},
  "image": {"image": "docker.io/library/nginx:latest"},
  "log_path": "nginx.0.log",
  "linux": {}
}
EOF

<<EOF cat > busybox.json
{
  "metadata": {"name": "busybox"},
  "image": {"image": "docker.io/library/busybox:latest"},
  "command": ["top"],
  "log_path": "busybox.0.log",
  "linux": {}
}
EOF

# execute them
crictl pull docker.io/library/busybox:latest
crictl pull docker.io/library/nginx:latest
crictl image

POD_ID=$(crictl runp pod.json)
echo ${POD_ID}
crictl pods --namespace default

CONTAINER_ID=$(crictl create ${POD_ID} busybox.json pod.json)
echo ${CONTAINER_ID}
crictl ps -a

crictl start ${CONTAINER_ID}
crictl exec -it ${CONTAINER_ID} sh
hostname
exit

# delete them
crictl ps -a
crictl stop ${CONTAINER_ID}
crictl rm ${CONTAINER_ID}
crictl ps -a
crictl pods
crictl stopp ${POD_ID}
crictl rmp ${POD_ID}
crictl pods
```

### Kubeletのセットアップ

```shell:worker-N
# kubeletのkubeconfig設定
(
  cd /etc/kubernetes
  export KUBECONFIG=/etc/kubernetes/kubelet.conf
  HOSTNAME=$(hostname)
  APISERVER=${HOSTNAME/worker/control}
  kubectl config set-cluster thecluster --certificate-authority=/etc/kubernetes/pki/ca.crt --server=https://${APISERVER}.internal:6443
  kubectl config set-credentials theuser --client-certificate=pki/${HOSTNAME}.crt --client-key=pki/${HOSTNAME}.key
  kubectl config set-context default --cluster=thecluster --user=theuser
  kubectl config use-context default
)

# kubeletの設定ファイル作成
# ref: https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/
<<EOF sed -e "/^ *#/d" >/etc/kubernetes/kubelet.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
serializeImagePulls: false
evictionHard:
    memory.available:  "100Mi"
    nodefs.available:  "10%"
    nodefs.inodesFree: "5%"
    imagefs.available: "15%"
    imagefs.inodesFree: "5%"
containerRuntimeEndpoint: /var/run/crio/crio.sock
tlsCertFile: "/etc/kubernetes/pki/$(hostname).crt"
tlsPrivateKeyFile: "/etc/kubernetes/pki/$(hostname).key"
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/etc/kubernetes/pki/ca.crt"
authorization:
  mode: Webhook
EOF

: escaped <<_____
clusterDomain: "cluster.local"
clusterDNS:
  - "$(sed -ne "/nameserver/{s/.* //;p}" </etc/resolv.conf)"
podCIDR: "192.168.64.128/25"
runtimeRequestTimeout: "15m"
resolvConf: "/run/systemd/resolve/resolv.conf"
_____

# DNS設定
ln -s /run/systemd/resolve/{stub-,}resolv.conf


# Create the kubelet.service systemd unit file:
<<EOF sed -e "/^ *#/d" >/etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=crio.service
Requires=crio.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/etc/kubernetes/kubelet.yaml \\
  --kubeconfig=/etc/kubernetes/kubelet.conf
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# kubeletの起動
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
```

### 動作確認

```shell:jumpbox
kubectl run busybox --image docker.io/library/busybox:latest -- top -b -d 30
# pod/busybox created
kubectl get pods
# NAME      READY   STATUS    RESTARTS   AGE
# busybox   1/1     Running   0          21s
kubectl logs busybox
# Mem: 944332K used, 71676K free, 164K shrd, 22444K buff, 779424K cached
# CPU:  0.0% usr  0.0% sys  0.0% nic  100% idle  0.0% io  0.0% irq  0.0% sirq
# Load average: 0.01 0.03 0.00 1/107 1
#   PID  PPID USER     STAT   VSZ %VSZ CPU %CPU COMMAND
#     1     0 root     R     4120  0.4   3  0.0 top -b -d 30
```

### Kube-proxyのセットアップ

```shell:worker-N
# TODO: server with balancer

# kube-proxyのkubeconfig設定
(
  cd /etc/kubernetes
  export KUBECONFIG=/etc/kubernetes/proxy.conf
  HOSTNAME=$(hostname)
  APISERVER=${HOSTNAME/worker/control}
  kubectl config set-cluster thecluster --certificate-authority=pki/ca.crt --server=https://${APISERVER}.internal:6443
  kubectl config set-credentials theuser --client-certificate=pki/kube-proxy.crt --client-key=pki/kube-proxy.key
  kubectl config set-context default --cluster=thecluster --user=theuser
  kubectl config use-context default
)


# Create the kube-proxy.service systemd unit file:
<<EOF sed -e "/^ *#/d" >/etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --kubeconfig /etc/kubernetes/proxy.conf \\
  # TODO FIXME
  --cluster-cidr 172.17.0.0/16 \\
  --proxy-mode iptables \\
  --nodeport-addresses primary \\

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
```


## dnsプラグインを導入する



## 確認

```shell:jumpbox
kubectl create deployment nginx --image=docker.io/library/nginx
#deployment.apps/nginx created

kubectl expose deployment nginx --port 80 --type NodePort
# service/nginx exposed

kubectl get all -o wide
# NAME                         READY   STATUS    RESTARTS   AGE     IP           NODE       NOMINATED NODE   READINESS GATES
# pod/nginx-5f789b8fdf-t24rp   1/1     Running   0          2m57s   172.17.0.2   worker-0   <none>           <none>
# 
# NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE     SELECTOR
# service/kubernetes   ClusterIP   172.20.0.1      <none>        443/TCP        6m13s   <none>
# service/nginx        NodePort    172.20.183.75   <none>        80:30090/TCP   22s     app=nginx
# 
# NAME                    READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES                    SELECTOR
# deployment.apps/nginx   1/1     1            1           2m57s   nginx        docker.io/library/nginx   app=nginx
# 
# NAME                               DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES                    SELECTOR
# replicaset.apps/nginx-5f789b8fdf   1         1         1       2m57s   nginx        docker.io/library/nginx   app=nginx,pod-template-hash=5f789b8fdf

curl --head http://127.0.0.1:8080 | head -1

kubectl port-forward nginx-5f789b8fdf-ghlpn 8080:80 &
# [1]+ kubectl port-forward nginx-5f789b8fdf-ghlpn 8080:80 &
# Forwarding from 127.0.0.1:8080 -> 80
# Forwarding from [::1]:8080 -> 80

curl --head http://127.0.0.1:8080
# Handling connection for 8080
# HTTP/1.1 200 OK
# Server: nginx/1.29.1
# Date: Mon, 22 Sep 2025 13:24:49 GMT
# Content-Type: text/html
# Content-Length: 615
# Last-Modified: Wed, 13 Aug 2025 14:33:41 GMT
# Connection: keep-alive
# ETag: "689ca245-267"
# Accept-Ranges: bytes

kubectl logs nginx-5f789b8fdf-ghlpn        
# (いろいろ起動時ログのあとに)
# 127.0.0.1 - - [22/Sep/2025:13:24:49 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/8.14.1" "-"

kubectl exec -it nginx-5f789b8fdf-ghlpn -- nginx -v
# nginx version: nginx/1.29.1

kubectl expose deployment nginx --port 80 --type NodePort
# service/nginx exposed

kubectl get svc nginx
# NAME    TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# nginx   NodePort   172.18.0.153   <none>        80:31000/TCP   81s

kubectl get nodes -o wide
# NAME       STATUS   ROLES    AGE    VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                       KERNEL-VERSION   CONTAINER-RUNTIME
# worker-0   Ready    <none>   144m   v1.34.0   192.168.64.13   <none>        Debian GNU/Linux 13 (trixie)   6.12.28          cri-o://1.34.0

curl --head 192.168.64.13:31000
# HTTP/1.1 200 OK
# Server: nginx/1.29.1
# Date: Mon, 22 Sep 2025 13:49:48 GMT
# Content-Type: text/html
# Content-Length: 615
# Last-Modified: Wed, 13 Aug 2025 14:33:41 GMT
# Connection: keep-alive
# ETag: "689ca245-267"
# Accept-Ranges: bytes

```

### metrics server

clusterDNSが必要
cf: github.com/kubernetes-incubator/metrics-server.git

kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl top node
kubectl top pod

## appendix

```
: /usr/lib/systemd/system/kubelet.service
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=https://kubernetes.io/docs/
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/bin/kubelet
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```
/etc/containers/registries.conf.d/registries.conf
 で unqualified-search-registries を一つにする
 short-name-mode=disabled にする
 auto-reload-registries をつける
```

### TODO

- ネットワーキング。ClusterIPとか
```
kube-apiserver:
  --service-cluster-ip-range=172.20.0.0/16 CIDR for svc
kube-controller-manager:
  --allocate-node-cidrs
  --cluster-cidr=172.17.0.0/16 CIDR for pods
  --service-cluster-ip-range=172.20.0.0/16 CIDR for svc
kubenet:
  --pod-cidr=172.17.0.0/16 CIDR for pod; only used in standalone mode
kube-proxy:
  --cluster-cidr=172.17.0.0/16 CIDR for pod
cni-plugin:
  ipam.subnet: CIDR for pod; used by cri-o

- サービスアカウントってなに
- coreDNS: cluster-cidr(pod-cidr) の　.10 を使う
- meticsAPI

apt-get install net-tools
for i in $(seq 0 2); do
  [ $(hostname) != worker-$i ] && \
    route add -net 172.17.$i.0/24 gw worker-$i.internal;
done


kubectl debug -it -n kube-system --image=docker.io/debian -c debugger3 coredns-5787d4c677-svjd4 --profile=general --target coredns


root@control-0:/# journalctl -u kube-controller-manager

/etc/systemd/system/kube-controller-manager.service

  --client-ca-file=/etc/kubernetes/pki/ca.crt \
  --requestheader-client-ca-file=/etc/kubernetes/pki/ca.crt \
  --root-ca-file=/etc/kubernetes/pki/ca.crt \
  --service-account-private-key-file=/etc/kubernetes/pki/sa.key \
```