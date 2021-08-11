---
title: "MikanOS Day0: Hello.EFI を Virtualboxで動かす"
emoji: "🍊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["MikanOS","UEFI"]
published: true
---

# この記事はなに

書籍「[ゼロからのOS自作入門](https://www.amazon.co.jp/gp/product/B08Z3MNR9J/ref=as_li_tl?ie=UTF8&camp=247&creative=1211&creativeASIN=B08Z3MNR9J&linkCode=as2&tag=matobaa-22&linkId=d89dcfcc3e4025954c3463250daf5ca5)」の 1.1「ハローワールド」を Windows10上のVirtualBoxで動作させたときのメモです。

書籍付録Aによれば、母艦がWindowsの場合、WindowsのWSL上にUbuntuを用意し、Ubuntu上にQEMU実行環境を用意し、母艦側のWindowsにXサーバを用意して、QEMUの画面を母艦側のXサーバで表示する、という手順になっていました。
Windows上にXサーバをインストールするのはちょっとなーと思ったので、GUI画面も持っている仮想環境を使う方法を探しました。

# VM実行環境を選択・導入

## Hyper-V を使う → ボツ

まず、Hyper-Vを使う方法を試しました。「Windowsの機能の有効化または無効化」でHyper-Vにチェックを入れて再起動し、Hyper-VマネージャからUbuntuテンプレートを使って仮想マシンを作成すれば使えるはずです。

ですが、Hyper-Vを有効化すると、Windowsの起動が目に見えて遅くなったので、ボツ、です。 Hyper-V は Windows Pro 以上じゃないと使えなかったような記憶もあり。

## VMWare workstation Player を使う → 優先度低

[VMware workstation Player](https://www.vmware.com/jp/products/workstation-player/workstation-player-evaluation.html)を使うと、Windows または Linux PC 上で 1 台の仮想マシンを実行できます。
「学生や非営利組織および個人の利用の場合は、無償バージョンが利用できます。商業組織で使用するには、商用ライセンスが必要です」とのことなので、劣後します。

## Virtualbox を使う

ライセンス的に使いやすい [Virtualbox](https://www.virtualbox.org/) を使うことにします。  
https://www.virtualbox.org/ からダウンロードし、インストールウィザードの指示どおりに導入します。

# 仮想ディスクを用意する

Virtualbox では仮想ディスクとして vdi, vhd, vmdk, hdd, qcow, qed といった形式が使えます。Windows上では、「コンピュータの管理」→「記憶域」→「ディスクの管理」で、VHD形式の仮想ディスクを作れるので、これで作成することにします。
サイズは適当に20GB、フォーマットをVHD、可変容量を選んでOKすると、vhdファイルができ、接続された状態になります。

作成された「ディスク1」は初期化されていないので、「初期化されていません」を右クリックして「ディスクの初期化」を選択。パーティションスタイルはよくわからないのでGPTのままOKをクリック。

「未割り当て」を右クリックして、「新しいシンプルボリューム」を選んで「次へ」「次へ」「次へ」。「このボリュームを次の設定でフォーマットする」に対して「FAT32」を選んで「次へ」「完了」。自動的にドライブレターが割り振られてエクスプローラーで開く。

P.30の手順に従って `EFI`フォルダを掘り、`BOOT`フォルダを掘り、そこに BOOTX64.EFI を置く。

ここまでできたら、「ディスクの管理」でディスク1を右クリックして「VHDを切断」したあと、「ディスクの管理」アプリは終了しておく。


# Virtualboxで仮想マシンを作る

Virtualboxを起動し、メニューの「仮想マシン」→「新規」から仮想マシンを作る。[Hello.EFI](https://github.com/uchan-nos/mikanos-build/blob/master/day01/bin/hello.efi)が動けばいいので、設定はとりあえず適当に。

  - 名前: MikanOS
  - タイプ: Other
  - バージョン: Other/Unknown(64bit)
  - メモリ: 64MB
  - ハードディスク: すでにある仮想ハードディスクファイルを使用する → 右端の黄色いボタンをクリックし、「追加」で先ほど作成したVHDファイルを追加後、それを選択する。

このまま起動すると `FATAL: INT18: BOOT FAILURE` となるので、起動する前に、
仮想マシン「MikanOS」の「設定」→「システム」→「拡張機能」→「EFIを有効化」にチェックしておく。

# 起動してみる

起動した! 「Hello World!」が出た!

```
BdsDxe: failed to load Boot0001 "UEFI VBOX CD-ROM VB2-01700376 " from PciBoot(0x0)/Pci(0x1,0x1)/Ata(Secondary,Master,0x0): Not Found
BdsDxe: loading Boot0002 "UEFI VBOX HARDDISK VBbbc753ae-9e77570b " from PciBoot(0x0)/Pci(0x1,0x1)/Ata(Primary,Master,0x0)
BdsDxe: starting Boot0002 "UEFI VBOX HARDDISK VBbbc753ae-9e77570b " from PciBoot(0x0)/Pci(0x1,0x1)/Ata(Primary,Master,0x0)
Hello, World!
```

