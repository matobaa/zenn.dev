---
title: "apple container で kubernetes を建てる (kubeadm; calico)"
emoji: "⛵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes", "k8s", "kubeadm", "calico", "applecontainer"]
published: true
---

## この記事はなに

Apple Container 上に kubeadm で Kubernetes を建てました。そのときの手順メモです。2026-08-14時点。

∵ Kubernetesのネットワーク周りの理解がイマイチで復習したかったため。NetworkPolicyを使いたいのでCNIはCalicoを使います。


## 手順と成果物の全体像

1. [Apple Container - Initial install](https://github.com/apple/container#initial-install)
1. [Bootstrapping clusters with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
    1. [cri-o](https://github.com/cri-o/packaging#distributions-using-deb-packages)
    1. [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
    1. [Calico quickstart guide](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)
1. [Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)


## デザイン

設定は以下とします:

- 母艦はm4mac 192.168.10.113/24 である (∵所与)
- apple container で建つVMのCIDRは 192.168.64.0/24 とする (∵既定値)
- Kubernetes は v1.36.3 を建てる (∵執筆時最新)
- 構築には kubeadm を利用する (∵方針)
- HA構成とするために、前段にロードバランサを一台、後段に Control Plane を3台、さらに Worker Node を3台建てる。
- etcdの配置は [Stacked Layout](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/#stacked-etcd-topology) とする (∵方針)
- CRIには [cri-o](https://cri-o.io/) 1.36 を利用する (∵方針)
- [CNI](https://www.cni.dev/)には [Calico](https://www.tigera.io/project-calico/) を利用する (∵方針; 注:Pod CIDRを既定値から変更する必要あり) ^[TODO:CKSに向けてCiliumに変えたい]
- Kubernetes の Service CIDR は `10.96.0.0/12` とする (∵デファクト)
- Kubernetes の Pod CIDR は `10.244.0.0/16` とする (∵デファクト)

<details><summary>ゴールイメージ</summary>
<img src="kubeadm_with_apple_container_0.drawio.svg" width=600px />
</details><br/>


# 構築手順

## 1. Apple Containerを導入する

[Apple Container - Initial install](https://github.com/apple/container#initial-install) をなぞって
[container-1.1.0-installer-signed.pkg](https://github.com/apple/container/releases/download/1.1.0/container-1.1.0-installer-signed.pkg) をインストールします。

執筆時最新は[1.2.2](https://github.com/apple/container/releases/tag/1.2.2)ですが、
ここでは [1.1.0](https://github.com/apple/container/releases/tag/1.1.0) を導入します。
(∵ [#2031](https://github.com/apple/container/issues/2041)@1.2.0〜1.2.2)

```zsh:m4mac
open container-1.1.0-installer-signed.pkg
```

Apple container のリゾルバで名前解決できるように設定します。

```zsh:m4mac
# container dns (192.168.64.1:53) で返すコンテナ名のドメインを設定する：
mkdir -p ~/.config/container
<<EOF tee ~/.config/container/config.toml
[dns]
domain = "container.internal"
EOF

# 上記 config.toml を読ませる
container system stop || true
container system start

# /etc/resolver/containerization.container.intenal を作ることで、
# mac上で dns resolverを container apiserver (127.0.0.1:2053) に向ける
sudo container system dns create container.internal
```

参考:
- [Set up DNS-based container names](https://github.com/apple/container/blob/main/docs/networking.md#set-up-dns-based-container-names)
- [Customize container default configuration values](https://github.com/apple/container/blob/main/docs/tutorials/container-system-config-tutorial.md)
- [アンインストール手順](https://github.com/apple/container#uninstall)


## 2. ノード群を起動する

systemd, cri-o, kubectl, kubeadm, kubectl が入ったコンテナイメージを作成し、それを使ってノードを起動します。

ノードを起動しておくことで、apple container のリゾルバでその名前を解決できるようになります。

<details><summary>Dockerfile</summary>

```Dockerfile:Dockerfile
# Dockerfile for systemd, cri-o, kubelet kubeadm kubectl
FROM debian:13-slim as systemd
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

FROM systemd as crio
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

```zsh:m4mac
# build
container build --target systemd --tag systemd  --file Dockerfile
container build --target crio    --tag k8s_node --file Dockerfile

# run them
container run -d --cap-add ALL --memory 2G --name control-0 k8s_node
container run -d --cap-add ALL --memory 2G --name control-1 k8s_node
container run -d --cap-add ALL --memory 2G --name control-2 k8s_node
container run -d --cap-add ALL --memory 2G --name worker-0 k8s_node
container run -d --cap-add ALL --memory 2G --name worker-1 k8s_node
container run -d --cap-add ALL --memory 2G --name worker-2 k8s_node
```

## 3. 前段にロードバランサを配置する

まず、前段に `HAProxy` を 1 台建てます。`k8s-api.container.internal` を API エンドポイントとして名乗らせ、その 6443/tcp を後段の Control Plane の 6443/tcp に流します。TLS は Control Plane 側で終端させることにします。

Apple Container ではコンテナの IP が変わるため、DNS 名で書くように統一します。

```zsh:m4mac
# LB用のコンテナを起動する
container run -d --cap-add ALL --memory 512M --name k8s-api systemd
container exec -it k8s-api bash
```

```bash:k8s-api
apt-get update && apt-get install -y haproxy

<<EOF tee /etc/haproxy/haproxy.cfg
frontend kubernetes_frontend
    bind *:6443
    mode tcp
    option tcplog
    default_backend kubernetes_backend

backend kubernetes_backend
    mode tcp
    balance roundrobin
    option tcp-check
    server control-0 control-0.container.internal:6443 check fall 2 rise 2
    server control-1 control-1.container.internal:6443 check fall 2 rise 2
    server control-2 control-2.container.internal:6443 check fall 2 rise 2
EOF

systemctl enable --now haproxy
```


## 4. 1台目のControlPlaneを作成する

`control-0`に入り、[Steps for the first control plane node](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#steps-for-the-first-control-plane-node) をなぞって、
`kubeadm init` します:

`--control-plane-endpoint` に LB の名前 `k8s-api.container.internal:6443` を指定することで、Apple Container の IP 変動に備えます。
https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#keepalived-and-haproxy


```bash:m4mac
container exec -it control-0 bash
```

```bash:control-0
# at control-0:
kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --control-plane-endpoint k8s-api.container.internal:6443 \
  --upload-certs

mkdir -p ~/.kube
cp -i /etc/kubernetes/admin.conf ~/.kube/config

# 出力されている join command を控えておく (あとで使う)
```

## 5. Calico を導入する

[Calico quickstart guide](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart) をなぞって、CNI Plugin である Calico をインストールします:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/v1_crd_projectcalico_org.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/tigera-operator.yaml

curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/custom-resources.yaml
sed -Ei "/cidr/s#:.+#: 10.244.0.0/16#" custom-resources.yaml
kubectl create -f custom-resources.yaml

kubectl get tigerastatus -w
```

calico-nodeが `Error: path "/sys/fs" is mounted on "/sys" but it is not a shared mount` を吐くので、全参加ノードの設定を修正します ^[/をsharedにするのは大丈夫なのか?]。

```bash:
/usr/bin/mount --make-shared /
/usr/bin/mount --make-shared /sys
```

再起動時にも有効化するために、カスタムsystemdサービスを仕込んでおきます:
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


## 6. 2台目以降のControlPlaneを作成する

`control-1`に入り、[Steps for the rest of the control plane nodes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#steps-for-the-rest-of-the-control-plane-nodes) をなぞって、
1台目の kubeadm init の結果を参照するか新規トークンを作成し、`kubeadm join` します:

```bash:既設のControlplaneで
kubeadm token create --print-join-command
```

```bash:新規参加するノードで
kubeadm join k8s-api.container.internal:6443 --token p1vryh.ltwnr9nwko200vos \
    --discovery-token-ca-cert-hash sha256:09a39d9ad9089d1b24f59a7280fbe86ab4ddba9589c1300ce02a51d26e0e5810 \
    --control-plane --certificate-key 01e6eb5041d8ad66f6081a6bf61531939e832d0a54ded6a36151d3feafe5bd9b
```

`container-2` でも同様にします。


## 7. Workerを作成する

`worker-0` に入り、[Install workers](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#install-workers)をなぞって、
1台目の kubeadm init の結果を参照するか新規トークンを作成し、`kubeadm join` します:

```bash:既設のControlplaneで
kubeadm token create --print-join-command
```

```bash:新規参加するノードで
kubeadm join k8s-api.container.internal:6443 --token p1vryh.ltwnr9nwko200vos \
	--discovery-token-ca-cert-hash sha256:09a39d9ad9089d1b24f59a7280fbe86ab4ddba9589c1300ce02a51d26e0e5810 
```

`work-1`, `worker-2` でも同様にします。

## 8. 動作確認する

```
kubectl create deployment hello --image=hashicorp/http-echo
kubectl create deployment hello --image=containous/whoami
kubectl scale deployment/hello --replicas=3
kubectl expose deployment hello --port 80
kubectl expose deployment hello --port 80 --type nodePort --name hellonp

# from node to worker nodeport
NODEPORT=$(kubectl get svc hellonp -o jsonpath={.spec.ports[].nodePort})
curl worker-0.container.internal:${NODEPORT}

# from neighbour pod
kubectl run netshoot --rm -it --image=nicolaka/netshoot
curl hello.default.svc.cluster.local

# sidecar
POD=$(kubectl get pods -l app=hello -o jsonpath={.items[0].metadata.name})
kubectl debug ${POD} -it --image=nicolaka/netshoot
```

おつかれさまでした。

## 残課題

1. kube-proxyとcalicoが競合しているのでは? https://oneuptime.com/blog/post/2026-03-13-calico-ebpf-common-mistakes/view#mistake-1-not-disabling-kube-proxy-before-enabling-ebpf
1. Cilium化: https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#k8s-quick-install
1. /をsharedにするのはまずくないか? /sys/fs/bpf だけ見せればいいのでは? https://oneuptime.com/blog/post/2026-03-13-calico-ebpf-common-mistakes/view#mistake-3-bpf-filesystem-not-mounted