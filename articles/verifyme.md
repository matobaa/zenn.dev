---
title: "この記事は確かに私が書きました"
emoji: "🪪"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: [x509]
published: false
---

### この記事の内容

- この記事は確かに自分が書いた、ということを説明するにはどうしたらいいかを考えてみた
- このURLは確かに自分が作ったということを説明できればいいはず
- "https://zenn.dev/matobaa/articles/verifyme" というURLに、自分の秘密鍵で署名したものを置いておけばいいのでは
- やってみた

### 手順

1. x509証明書 {秘密鍵 + 証明書} を用意する
2. 秘密鍵を用いて対象文字列を署名する
3. 対象文字列、署名、証明書を公開する
4. 利用者として署名を検証する

### 1. x509証明書 {秘密鍵 + 証明書} を用意する

ご用意しておきました

### 環境準備

今回は Windows 11 Pro 22H2 64bit を用いました。
- https://github.com/jpki/myna/releases から myna-0.5.1-windows-x64.zip をダウンロードして展開
- 中身の myna.exe を取り出して、任意の作業用フォルダに配置する
- コマンドプロンプトを起動し、作業用フォルダに移動する

### 2. 秘密鍵を用いて対象文字列を署名する

`myna jpki cms sign --form pem --pin P1N0F51GNC3RT --in input.txt --out out.txt`

### 3. 対象文字列、署名、証明書を公開する

対象文字列
`https://zenn.dev/matobaa/articles/verifyme`

:::details 署名
```
```
:::

:::details 署名用公開鍵
```
```
:::

### 4. 利用者として署名を検証する



