---
title: "[VS Code] 中ボタンでスクロールするには"
emoji: "↕️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode"]
published: false
---

# TL;DR (この記事はなに?)

Windowsのほとんどのアプリでは「中ボタン+mousemove」でスクロールできるのですが、Visual Studio Code だとマルチカーソル機能に横取りされてしまってスクロールできなくて不便です。検索したところ、インターネットの奥深くに解決方法があったので、自分用にメモを残しています。


# いきさつ

## 中ボタンスクロールのススメ

過去に腱鞘炎になったきっかけで、マウスからトラックボールに持ち換えた私ですが、そのときに入手した Trackman marble T-BC21 が大玉でとても使いやすく、今でも使い続けています。  
ですがこいつ、初代は1998年、Marbleも2008年発売という名機であり、ホイールがありません。4つのボタンはカスタマイズ可能なので、左右クリックのほかに「中ボタン」、つまり、ホイールを押し込んだ動作を割り当てています。Windows上の多くのアプリでは、ホイールを押し込んだままマウスを移動すると、その方向にスクロールできます。ホイールをくるくる回すよりスムーズにスクロールできるのでオススメです。

## VS Codeで使えなかった……

この機能が、VS Code では使えなくて、とても不便でした。ホイールがなく、中ボタンスクロールが使えないとなると、キーボードでCtrl-Up/Dnをタイプするか、スクロールバーにポインタを合わせてスクロールするしかないのです。マウス操作中にキーボードに持ち換えたり、スクロールバーにポインタを合わせたりするのは手間がかかり、思考がさえぎられてしまいます。

## 解決方法を見つけた!

これでは開発の生産性も上がりません。きっと解決できるプラグインがあるだろう、と探したのですが見つからず、VS CodeのIssueとして [#6302](https://github.com/microsoft/vscode/issues/6302)や [#104183](https://github.com/microsoft/vscode/issues/104183) を見つけましたが、直接的な解決にはつながりませんでした。  

しかし、そのIssue上で回避策が言及されていました。以下の3つです:

1. https://github.com/qooqu/vs-code-mbutton-scroll-ahk
2. https://github.com/DanielBoxer/VSCroll
3. https://github.com/UziTech/atom-scroll-editor-on-middle-click

上記のうち、VSCroll を試したところ、理想的に動作したので、やり方を紹介します。  
vs-code-mbutton-scroll-ahk も同じ操作感でした。atom-scroll-editor-on-middle-click は試していません。

# 導入方法

ahkスクリプトとして実現されていますので、まずAutoHotKeyを導入し、そのうえでスクリプトを実行する、という手順になります。すなわち:

## AutoHotKey を導入する

[AutoHotKey](https://www.autohotkey.com/) のWebサイトから、[Current Version](https://www.autohotkey.com/download/ahk-install.exe) をダウンロードして実行します。実行にあたって管理者権限などは不要です。

## スクリプトを実行する

[VSCroll.ahk](https://github.com/DanielBoxer/VSCroll/raw/main/VSCroll.ahk) を任意の場所にダウンロードし、ダブルクリックして実行します。タスクトレイにAutoHotKeyのアイコンが登場すれば有効化されています。  
[shell:startup](https://support.microsoft.com/ja-jp/windows/150da165-dcd9-7230-517b-cf3c295d89dd) に置いておけば起動時に有効化できます。

こんな感じで使えます:

![動画gif](https://user-images.githubusercontent.com/65575771/177434189-42c2942b-1919-4770-adab-ca9bf65f50ac.gif)

---- 

以上、よき開発ライフを!!