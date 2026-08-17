---
title: "apple container で kubernetes を建てる (kubeadm編)"
emoji: "⛵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## この記事はなに

Apple Container 上に kubeadm で Kubernetes を建てました。そのときの手順メモです。2026-08-14時点。
<br/>
∵ Kubernetesのネットワーク周りの理解がイマイチで、Calicoを復習したかったため。


## 手順と成果物の全体像

1. [Apple Container - Initial install](https://github.com/apple/container#initial-install)
1. [Bootstrapping clusters with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
    1. cri-o
    1. [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
    1. [Calico quickstart guide](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)
1. [Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)


## デザイン

設定は以下とします:

- 母艦はm4mac 192.168.10.113/24 である (∵所与)
- Host (=apple container) の CIDR は 192.168.64.0/24 とする (∵既定値)
- Kubernetes は 1.36.3 を建てる (∵執筆時最新)
- 構築には kubeadm を利用する (∵方針)
- HA構成とし、Control Plane: 2台 + Worker Node: 2台 を建てる。ただし API-Serverの前に外部ロードバランサは置かない (∵方針) ^[Knot DNS で roundrobin + RFC 2136 なDynDNS建てたい] 
- etcdの配置は [Stacked](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/#stacked-etcd-topology) とする (∵方針)
- CRIには [cri-o](https://cri-o.io/) 1.36 を利用する (∵方針)
- [CNI](https://www.cni.dev/)には [Calico](https://www.tigera.io/project-calico/) を利用する (∵方針; 注:Pod CIDRを既定値から変更する必要あり) ^[TODO:CKSに向けてCiliumに変える]
- Kubernetes の Service CIDR は `10.96.0.0/12` とする (∵デファクト)
- Kubernetes の Pod CIDR は `10.244.0.0/16` とする (∵デファクト)
- Serv

<details><summary>ゴールイメージ</summary>
<img src="kubeadm_with_apple_container_0.drawio.svg" width=600px />
</details><br/>

# 構築手順


## 1. Apple Containerを導入する

[Apple Container - Initial install](https://github.com/apple/container#initial-install) をなぞって
[container-1.1.0-installer-signed.pkg](https://github.com/apple/container/releases/download/1.1.0/container-1.1.0-installer-signed.pkg) をインストールします。
<br/>
執筆時最新は[1.2.2](https://github.com/apple/container/releases/tag/1.2.2)ですが、
ここでは [1.1.0](https://github.com/apple/container/releases/tag/1.1.0) を導入します。
(∵ [#2031](https://github.com/apple/container/issues/2041)@1.2.0〜1.2.2)

container環境設定(初回のみ)
```
container system stop

# container dns (192.168.64.1:53) で返すコンテナ名のドメインを設定
mkdir -p ~/.config/container
<<EOF tee ~/.config/container/config.toml
[dns]
domain = "container.internal"
EOF

# /etc/resolver/containerization.container.intenal を作る = mac上で dns resolverを container apiserver (127.0.0.1:2053) に向ける
container system start
sudo container system dns create container.internal
```

参考:
- [Set up DNS-based container names](https://github.com/apple/container/blob/main/docs/networking.md#set-up-dns-based-container-names)
- [Customize container default configuration values](https://github.com/apple/container/blob/main/docs/tutorials/container-system-config-tutorial.md)
- [アンインストール手順](https://github.com/apple/container#uninstall)

## 寄り道 - HA構成用に前段にロードバランサを配置する

```bash:
container run -d --name k8s.container.internal --image ATDK
``` 


## 2. ノード群を起動する

以下の内容を`Dockerfile`に書き込み、イメージをビルドし、起動します:
<details><summary>Dockerfile</summary>

```Dockerfile:Dockerfile
# Dockerfile for systemd, cri-o, kubelet kubeadm kubectl
FROM debian:13-slim
# systemd
# ref. https://hub.docker.com/_/centos#dockerfile-for-systemd-base-image
# ref. https://github.com/apple/container/blob/main/docs/container-machine.md
RUN DEBIAN_FRONTEND=noninteractive apt-get update \
     && apt-get install -y systemd \
    && rm -rf /var/lib/apt/lists/* \
              /lib/systemd/system/multi-user.target.wants/* \
              /etc/systemd/system/*.wants/* \
              /lib/systemd/system/local-fs.target.wants/* \
              /lib/systemd/system/sockets.target.wants/*udev* \
              /lib/systemd/system/sockets.target.wants/*initctl* \
              /lib/systemd/system/basic.target.wants/* \
              /lib/systemd/system/anaconda.target.wants/*
CMD ["/lib/systemd/systemd"]

# cri-o, kubectl, kubeadm, kubectl
# ref. https://github.com/cri-o/packaging#distributions-using-deb-packages

ENV KUBERNETES_VERSION=v1.36
ENV CRIO_VERSION=v1.36
RUN apt-get update && apt-get install -y gpg curl vim procps

RUN curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key \
    | gpg --batch --yes --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg \
    && echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] " \
            "https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" \
       | tee /etc/apt/sources.list.d/kubernetes.list

RUN curl -fsSL https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key \
    | gpg --batch --yes --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg \
    && echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] " \
            "https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/ /" \
       | tee /etc/apt/sources.list.d/cri-o.list

RUN apt-get update && apt-get install -y cri-o kubelet kubeadm kubectl
RUN echo net.ipv4.ip_forward = 1 | tee /etc/sysctl.d/k8s.conf
RUN systemctl enable crio.service
```

</details><br/>

```bash
# build
container build --tag k8s_node --file Dockerfile

# run them
container run -d --cap-add ALL --memory 2G --name control-0 k8s_node
container run -d --cap-add ALL --memory 2G --name control-1 k8s_node
container run -d --cap-add ALL --memory 2G --name worker-0 k8s_node
container run -d --cap-add ALL --memory 2G --name worker-1 k8s_node
```

```bash
# machine を利用する場合は
container machine create --home-mount none --memory 2G --name control-0 k8s_node
container machine run -it --root -w /root -n control-0 bash
```

## 1台目のControlPlaneを作成する

`control-0`に入り、[Steps for the first control plane node](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#steps-for-the-first-control-plane-node) をなぞって、
`kubeadm init` します:

TODO: 高可用性にするため、DNSにAPIエンドポイントを記入する  
` --control-plane-endpoint kinaco.local:6443 `  
https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#keepalived-and-haproxy

```bash:m4mac
container exec -it control-0 bash
```

```bash:control-0:
# at control-0:
echo 1 > /proc/sys/net/ipv4/ip_forward
sysctl -w net.ipv4.ip_forward=1

kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --control-plane-endpoint "$(hostname -i):6443" \
  --upload-certs

# 出力されている join command を控えておく (あとで使う)

mkdir -p ~/.kube
cp -i /etc/kubernetes/admin.conf ~/.kube/config
```

## Calico を導入する

[Calico quickstart guide](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart) をなぞって、CNI Plugin である Calico をインストールします:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/v1_crd_projectcalico_org.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/tigera-operator.yaml

curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/custom-resources.yaml
sed -Ei "/cidr/s#:.+#: 10.244.0.0/16#" custom-resources.yaml
kubectl create -f custom-resources.yaml
kubectl get tigerastatus -w


```

calico-nodeが `Error: path "/sys/fs" is mounted on "/sys" but it is not a shared mount` を吐くので、全参加ノードの設定を修正する:
^[/をsharedにするのは大丈夫なのか?]

```bash:
/usr/bin/mount --make-shared /
/usr/bin/mount --make-shared /sys
```

再起動時にも有効化するために、カスタムsystemdサービスを仕込んでおく:
```bash:
<<EOF tee /etc/systemd/system/container-mount-propagation.service
[Unit]
Description=Ensure mount propagation is shared for container runtimes
Before=crio.service kubelet.service
After=local-fs.target
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/mount --make-shared /
ExecStart=/usr/bin/mount --make-shared /sys
#ExecStart=/usr/bin/mount --make-shared /sys/fs/bpf
#ExecStart=/usr/bin/mount --make-shared /var/run/calico

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable container-mount-propagation.service
systemctl restart container-mount-propagation.service
findmnt -o TARGET,PROPAGATION /sys
: expected result:
: TARGET PROPAGATION
: /sys   shared
```


## 2台目のControlPlaneを作成する

`control-1`に入り、[Steps for the rest of the control plane nodes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#steps-for-the-rest-of-the-control-plane-nodes) をなぞって、
1台目の kubeadm init の結果を参照するか新規トークンを作成し、`kubeadm join` します:

```bash:既設のControlplaneで
kubeadm token create --print-join-command
```

```bash:新規参加するノードで
echo 1 > /proc/sys/net/ipv4/ip_forward

kubeadm join 192.168.64.13:6443 --token p1vryh.ltwnr9nwko200vos \
    --discovery-token-ca-cert-hash sha256:09a39d9ad9089d1b24f59a7280fbe86ab4ddba9589c1300ce02a51d26e0e5810 \
    --control-plane --certificate-key 01e6eb5041d8ad66f6081a6bf61531939e832d0a54ded6a36151d3feafe5bd9b

kubectl get tigerastatus
```

## Workerを2台作成する

```bash:既設のControlplaneで
kubeadm token create --print-join-command
```

```bash:新規参加するノードで
echo 1 > /proc/sys/net/ipv4/ip_forward
kubeadm join 192.168.64.13:6443 --token p1vryh.ltwnr9nwko200vos \
	--discovery-token-ca-cert-hash sha256:09a39d9ad9089d1b24f59a7280fbe86ab4ddba9589c1300ce02a51d26e0e5810 
```


## 動作確認する

ATDK

## 残課題

1. ~~DNSが両立していない~~ → calicoが完全に起動したあとは解消した
    - container内だと container DNSである 192.168.64.1#53 が /etc/resolv.conf に書かれていて、example.com. を解決できるが、kubernetes.default.svc.cluster.internal を解決できない。明示的に 
    - pod内だと、kube-dns である 10.244.0.10#53
1. echo 1 > /proc/sys/net/ipv4/ip_forward を永続化したい。sysctl
1. kube-proxyとcalicoが競合しているのでは? https://oneuptime.com/blog/post/2026-03-13-calico-ebpf-common-mistakes/view#mistake-1-not-disabling-kube-proxy-before-enabling-ebpf
1. Cilium化: https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#k8s-quick-install
1. /をsharedにするのはまずくないか? /sys/fs/bpf だけ見せればいいのでは? https://oneuptime.com/blog/post/2026-03-13-calico-ebpf-common-mistakes/view#mistake-3-bpf-filesystem-not-mounted