---
title: "自作PDFのXREFテーブルを生成する"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [pdf]
published: true
---

PDFファイルを自作するにあたり、各オブジェクトが格納されているバイトオフセットの一覧を格納した XREF テーブルを作りたい。
テキストエディタでPDFを編集していると次第に XREF がずれていってエラーになってしまうので、このコマンドを使って再生成してやる。

## TL;DR (この記事はなに)

- PDFファイルの XREFテーブルやTrailerのstartxrefに記載するバイトオフセットは `grep -b` で取れるよ
- それをawkで加工してXREFテーブルを生成したよ

(注: Linearized PDF には未対応です)

## PDF仕様解説: XREFセクションとは

```pdf:XREFセクションの例
xref
0 8
0000000000 65535 f
0000000010 00000 n
0000000095 00000 n
0000000154 00000 n
0000000230 00000 n
0000000439 00000 n
0000000560 00000 n
0000000593 00000 n
```

[PDFReference 3.4.3 Cross-Reference Table](https://www.adobe.com/content/dam/acom/en/devnet/pdf/pdfs/pdf_reference_archives/PDFReference.pdf#page=84) によれば:

- XREFテーブルは、PDFオブジェクトにランダムアクセスさせるための情報を保持する。
- `xref`行で始めて、サブセクションを並べる。
- サブセクションは数字を二つ並べた行で始める。連続したオブジェクトの最初の番号と、その個数。
- 続いて、クロスリファレンス情報を一行単位で記載する。各行は改行も含めて20バイト。
- 形式は `nnnnnnnnnn ggggg n<eol>`。バイトオフセット(10桁)、空白、オブジェクトの世代(5桁)、空白、`n`(=使用中フラグ)、改行文字(CR+LF)。
- そして、そのxrefセクションのバイトオフセットを、PDFファイル末尾の`startxref`で指し示す。

[Example G.2](https://www.adobe.com/content/dam/acom/en/devnet/pdf/pdfs/pdf_reference_archives/PDFReference.pdf#page=780) を見るとイメージをつかめると思う。

## XREFを生成するワンライナー

```shell-session
$ grep -b " obj" 1.pdf
10:1 0 obj
150:2 0 obj
235:3 0 obj
295:4 0 obj
372:5 0 obj
589:6 0 obj

$ cat 1.pdf | grep -b " obj" | awk -F: '{print $2, $1}' | sort -n | \
  awk 'BEGIN {printf("0000000000 65535 f\n")} {printf("%010d %05d n\n",$4,$2)}'
0000000000 65535 f
0000000010 00000 n
0000000150 00000 n
0000000235 00000 n
0000000295 00000 n
0000000372 00000 n
0000000589 00000 n

$ grep -b "^xref" 1.pdf
733:xref

$ cat 1.pdf | grep -b "^xref" | awk -F: '{print "startxref\n" $1 "\n%%EOF"}'
startxref
701
%%EOF
```

## おまけ: ストリームの長さが正しいか確認する

```shell-session
$ cat 1.pdf | grep -Ebn "stream|Length"
38:598:  << /Length 58 >>
39:618:stream
45:682:endstream

$ cat 1.pdf | grep -Ebn "stream|Length" | awk -F: '
  /\/Length/ {lineno=$1; match($0, /.Length ([0-9]+)/, len)}
  /:stream/ {begin=$2}
  /:endstream/ {
    if(len[1]!=$2-begin) {print lineno": expected " $2-begin " but actual " len[1]}}'
38: expected 64 but actual 58

$ vi +38 1.pdf
```