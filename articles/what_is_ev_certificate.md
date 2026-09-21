---
title: "EV証明書とは何なのか"
emoji: "🔏"
type: "idea"
topics: ["TLS","tls証明書","ssl","openssl",]
published: true
---

# この記事はなに

> **Let's Encrypt はビジネス利用に適しているのでしょうか。無料の認証局という理解でしかないので、偏見であればご指摘いただきたく。**

というお客様の質問をきっかけに、SSL/TLS証明書界隈を掘ったときのメモです。

EV証明書とは何なのか、DV証明書、OV証明書、IV証明書とは何がどう違うのか、について明らかにします。

## TL;DR (いきなりまとめ)

- EV証明書とは、SSL/TLS証明書のうち、その発行先法人の身分が社会的に確認できているものを EV証明書 (Extended validation) という。
- EV証明書には「証明書ポリシー」に固有値が埋め込まれていて、DV/OV/IV証明書と区別できる。
- 中間者攻撃や SASE/CASBによる SSL Bump では、EV証明書を作成できない。

# 以下コピペ (あとで整形する)

スラド https://security.srad.jp/story/23/10/09/0328248/ 経由、徳丸さんが「www.whitehouse.gov も Let's Encrypt」という情報を得た。
FirefoxもLetsEncryptを信頼しているとの情報もある。(FirefoxはOSトラストストアを利用してないし、日本政府のGKPIも信用しなかった)
https://free-ssl.jp/blog/2016-08-05.html

TLS証明書の見分け方を完全に理解した。
DV(ドメイン証明) := サブジェクトにCNを含む、Oを含まない
OV(組織証明) := サブジェクトにCN、Oを含む、SerialNumberを含まない
IV(個人証明) := サブジェクトにCN、OまたはSurNameを含む、SerialNumberを含まない
EV(拡張証明) := サブジェクトにCN、O、SerialNumberを含む
google.com は DV
microsoft.com は OV
apple.com は EV

antiphishing.jp はオリジナルサイトはEV証明だが、ZScalerを経由した場合、サブジェクトにSerialNumberを含んでいるのにChromeブラウザはEV証明扱いしていない。

antiphishing.jp について ZScalerありなしで比較してみた。
Subjectは完全一致した。
X509v3 Extensions に差がみられる。
ZScaler経由の場合:
- Basic Constraints: CA=false
- Subject Alternative Name: DNS=www.antiphishing.jp
- Key Usage: Digital Signature, Key Enciperment
- Extended Key Usage: TLS Web Server Authentication, TLS Web Client Authentication
- CRL Distribution Points: URI=
ZScalerを経由しない場合、上記に加えて以下があった:
- Certificate Policies: [1.2.392.200091.100.721.1, 2.23.140.1.1, 1.2.392.200081.1.22.1]
- Subject Key Identifier: blob[20]
- Authority Key Identifier: blob[20]
- Authority Information Access: (URLが二つ)
- CT Precertificate SCTs: (なんかいろいろ)

https://ja.wikipedia.org/wiki/Extended_Validation_証明書#EV証明書の特定 で曰く:
EV証明書を特定する一貫した方法は存在しない。
まじか!! 
openssl asn1parse してみたらこんな感じ。こんなの偽装できそうだよなぁ、Webブラウザに埋め込んだリテラルと比較してるのだろうか。
725:d=5  hl=2 l=   3 prim: OBJECT            :X509v3 Certificate Policies
  730:d=5  hl=2 l= 108 prim: OCTET STRING
      0000 - 30 6a 30 0c 06 0a 2a 83-08 8c 9b 1b 64 85 51 01   0j0...*.....d.Q.
      0010 - 30 07 06 05 67 81 0c 01-01 30 51 06 09 2a 83 08   0...g....0Q..*..
      0020 - 8c 9b 11 01 16 01 30 44-30 42 06 08 2b 06 01 05   ......0D0B..+...
      0030 - 05 07 02 01 16 36 68 74-74 70 73 3a 2f 2f 77 77   .....6https://ww
      0040 - 77 2e 63 79 62 65 72 74-72 75 73 74 2e 6e 65 2e   w.cybertrust.ne.
      0050 - 6a 70 2f 73 73 6c 2f 72-65 70 6f 73 69 74 6f 72   jp/ssl/repositor
      0060 - 79 2f 69 6e 64 65 78 2e-68 74 6d 6c               y/index.html

      SECOMTrustがChromiumに「ウチもEVにしてよ」というRFEを上げていた:
https://issues.chromium.org/issues/41230756#comment23
で、こんな感じでWebブラウザ(Chromium)に埋め込まれているところまで突き止めた。
なるほどー。OIDとsha256で検証してるのね。ここまでやってるなら偽装できないな。理解した
https://source.chromium.org/chromium/chromium/src/+/main:net/data/ssl/chrome_root_store/root_store.textproto;drc=8c9c569064da5038d88ae12f49171ff988e4dd98;l=520
 
Webブラウザ(Firefox)はここで発見:
https://hg.mozilla.org/mozilla-central/file/tip/security/nss/lib/ckfw/builtins/certdata.txt#l5834
https://hg.mozilla.org/mozilla-central/file/tip/security/certverifier/ExtendedValidation.cpp#l705

> Extv3に "1.2.392.200091.100.721.1" が存在したら、証明書のSHA256が
513B2CECB810D4CDE5DD85391ADFC6C2DD60D87BB736D2B521484AA47A0EBEF6であることを期待する

 | sed 1,/END/d | openssl x509 -ext subjectAltName,certificatePolicies,authorityInfoAccess -noout


echo | openssl s_client --connect www.antiphishing.jp:443 -showcerts 2>NUL| sed '1,/END/d' | openssl x509 -text -noout | sed -n "/CA Issuers/s/.*URI.//p"
http://repository.secomtrust.net/SC-Root2/SCRoot2ca.cer
echo | openssl s_client --connect www.antiphishing.jp:443 -showcerts 2>NUL| sed '1,/END/d' | openssl x509 -text -noout | sed -n "/CA Issuers/s/.*URI.//p" | xargs -i curl -s {} | openssl x509 -noout -fingerprint -sha256

curl http://repository.secomtrust.net/SC-Root2/SCRoot2ca.cer | openssl x509 -fingerprint -sha256 -noout
sha256 Fingerprint=51:3B:2C:EC:B8:10:D4:CD:E5:DD:85:39:1A:DF:C6:C2:DD:60:D8:7B:B7:36:D2:B5:21:48:4A:A4:7A:0E:BE:F6

