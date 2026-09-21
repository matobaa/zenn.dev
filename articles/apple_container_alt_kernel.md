---
title: "Apple Container のカーネルを置き換える"
emoji: "⛵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes", "k8s", "kubeadm", "ebpf", "calico", "cilium", "kiac", "applecontainer"]
published: true
---

既定値では　Kata Container のカーネルを使いますが、これを入れ替えます。

∵ 勉強用に xt_socket カーネルモジュールを使えるVMが欲しいため。


既定値

https://github.com/kata-containers/kata-containers/releases/download/3.28.0/kata-static-3.28.0-arm64.tar.zst
```
cat /proc/version
Linux version 6.18.15 (@40fb4c1548ae) (gcc (Ubuntu 11.4.0-1ubuntu1~22.04.3) 11.4.0, GNU ld (GNU Binutils for Ubuntu) 2.38) #1 SMP Tue Mar 17 01:36:53 UTC 2026
```

入れ替え先

https://kubesimplify.com/products/kiac
https://github.com/saiyam1814/kiac/releases?page=2#release-kernel-v6.12.28-full

```
cat /proc/version
Linux version 6.12.28-kiac-full (root@8c79b311311e) (gcc (Ubuntu 13.3.0-6ubuntu2~24.04.1) 13.3.0, GNU ld (GNU Binutils for Ubuntu) 2.42) #1 SMP Mon Jul  6 11:29:58 UTC 2026
```


## 入れ替え方法

```bash
# 入手
curl -L -O https://github.com/saiyam1814/kiac/releases/download/kernel-v6.12.28-full/kiac-kernel-6.12.28-full

# 設定
container system kernel set --binary ./kiac-kernel-6.12.28-full

# 確認
ls -l  ~/Library/Application\ Support/com.apple.container/kernels
container run --rm ubuntu cat /proc/version
```


