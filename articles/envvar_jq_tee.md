---
title: "jsonの複数の値を環境変数に設定するワンライナー"
emoji: "🐚"
type: "tech"
topics: ["aws","bash","jq"]
published: true
---

## この記事はなに

jsonに含まれる複数の値を一度に環境変数に設定するワンライナーを編んだメモです。

`aws sts assume-role` の結果を環境変数に格納することでスイッチロールできます。しかし、`aws sts assume-role`は実行するたびに応答が変わるので、その中の3つの値を取り出すには一時ファイルに保存したりする必要があってイマイチだったのを解決します。

## TL;DR

```
TARGET_ROLE=arn:aws:iam::123456789012:role/target_role_name

declare -x $(aws sts assume-role --role-session-name cli --role-arn ${TARGET_ROLE} | jq -r '.Credentials | "AWS_ACCESS_KEY_ID=\(.AccessKeyId) AWS_SECRET_ACCESS_KEY=\(.SecretAccessKey) AWS_SESSION_TOKEN=\(.SessionToken)" ' | tee /dev/stderr)
```

## 逐次解説

```
TARGET_ROLE=arn:aws:iam::123456789012:role/target_role_name

declare -x $(
    aws sts assume-role --role-session-name cli --role-arn ${TARGET_ROLE} \
    | jq -r '.Credentials | "AWS_ACCESS_KEY_ID=\(.AccessKeyId) AWS_SECRET_ACCESS_KEY=\(.SecretAccessKey) AWS_SESSION_TOKEN=\(.SessionToken)" ' \
    | tee /dev/stderr
)
```

### `TARGET_ROLE=arn:aws:iam::123456789012:role/target_role_name`

awsコマンドに与えるパラメータを外だしして、シェル変数^[正しくは「パラメータ」と呼ぶ]で定義しておきます。ワンライナーの再利用性を高めます。

### `declare -x`

シェル変数を定義するためのシェルのビルトインコマンドです。  
-x オプションをつけると export され、コマンドやサブシェルの実行時にも反映されます。exportコマンドでも同様に実現できますが、変数定義したい意図を明示するために declare を選択しました。

### `aws sts assume-role --role-session-name cli --role-arn ${TARGET_ROLE}`
aws CLI でロールを切り替えます。切り替え後のロールのための設定がjson形式で返ってきます。この実行結果を環境変数に設定することで、切り替え後のロールを利用できます。

応答の例:
```
{
    "Credentials": {
        "AccessKeyId": "AKIAIOSFODNN7EXAMPLE",
        "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
        "SessionToken": "L79BL2nkAZSNi9Z4PMnLKCDkFIbAKvucp5KHMWvzy9Y=",
        "Expiration": "2025-08-16T07:01:34+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROAIOSFODNN7EXAMPLE:cli",
        "Arn": "arn:aws:sts::123456789012:assumed-role/target_role/cli"
    }
}
```

### `jq -r '.Credentials | "AWS_ACCESS_KEY_ID=\(.AccessKeyID) ... "　'`

応答のjsonから、A=X B=Y C=Z のような形の文字列を組み立てています。

### `tee /dev/stderr`

組み立てた A=X B=Y C=Z のような文字列を、標準エラー出力にも複製しています。  
内容を確認する必要がなければ省略可能です。

### `$( ... )`

位置パラメータに、カッコ内のコマンドの実行結果を配置します。  
結果として、`declare -x A=X B=Y C=Z` のようなコマンドを実行することになります。

### まとめ

awsコマンドの実行結果を一時ファイルに保存したりしてイマイチだったのを解決できました。  
イイネ!と感じたら❤️クリックお願いします!