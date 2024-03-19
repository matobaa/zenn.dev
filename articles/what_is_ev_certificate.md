---
title: "EV証明書とは何なのか"
emoji: "🔏"
type: "idea"
topics: ["TLS","tls証明書","EV証明書","ssl","openssl"]
published: true
---

# この記事はなに

お客様の質問から以下のような質問を頂戴した:
> **Let's Encrypt はビジネス利用に適しているのでしょうか。無料の認証局という理解でしかないので、偏見であればご指摘いただきたく。**

これをきっかけに、SSL/TLS証明書界隈を掘ったときのメモです。

EV証明書とは何なのか、DV/OV/IV証明書とは何がどう違うのか、について明らかにします。

## TL;DR (いきなりまとめ)

- EV証明書とは、SSL/TLS証明書のうち、その発行先法人の身分が社会的に確認できているものを EV証明書 (Extended validation) という。
- EV証明書には「証明書ポリシー」に固有値が埋め込まれている。この固有値とハッシュを見ることで、EV証明書であることを確認できる。
- 中間者攻撃や SASE/CASBによる SSL Bump では、EV証明書を作成できない。

## 確認するサンプル

ここでは以下のサンプルについて確認する:
1. EV証明書の例として、https://www.apple.com/ および https://www.antiphishing.jp/
2. OV証明書の例として、https://www.microsoft.com/ および httos://www.alibaba.com/
3. DV証明書の例として、https://www.whitehouse.gov/ 

## 証明書を取得する方法

以下のコマンドで証明書を取得する:  
`echo | openssl s_client --showcerts --connect www.whitehouse.gov:443`

:::details 結果
```shell-session
CONNECTED(00000003)
depth=2 C = JP, O = "SECOM Trust Systems CO.,LTD.", OU = Security Communication RootCA2
verify return:1
depth=1 C = JP, O = "Cybertrust Japan Co., Ltd.", CN = Cybertrust Japan SureServer EV CA G3
verify return:1
depth=0 jurisdictionC = JP, serialNumber = 0100-05-006504, businessCategory = Private Organization, C = JP, ST = Tokyo, L = Chuo-ku, O = Japan Computer Emergency Response Team Coordination Center, CN = www.antiphishing.jp
verify return:1
---
Certificate chain
 0 s:jurisdictionC = JP, serialNumber = 0100-05-006504, businessCategory = Private Organization, C = JP, ST = Tokyo, L = Chuo-ku, O = Japan Computer Emergency Response Team Coordination Center, CN = www.antiphishing.jp
   i:C = JP, O = "Cybertrust Japan Co., Ltd.", CN = Cybertrust Japan SureServer EV CA G3
-----BEGIN CERTIFICATE-----
MIIHPTCCBiWgAwIBAgIUZb2OcOdR3VpxX2iY/PmiaoQolcQwDQYJKoZIhvcNAQEL
BQAwYTELMAkGA1UEBhMCSlAxIzAhBgNVBAoTGkN5YmVydHJ1c3QgSmFwYW4gQ28u
LCBMdGQuMS0wKwYDVQQDEyRDeWJlcnRydXN0IEphcGFuIFN1cmVTZXJ2ZXIgRVYg
Q0EgRzMwHhcNMjQwMTA1MDA1ODUxWhcNMjUwMTMxMTQ1OTAwWjCB3zETMBEGCysG
AQQBgjc8AgEDEwJKUDEXMBUGA1UEBRMOMDEwMC0wNS0wMDY1MDQxHTAbBgNVBA8T
FFByaXZhdGUgT3JnYW5pemF0aW9uMQswCQYDVQQGEwJKUDEOMAwGA1UECBMFVG9r
eW8xEDAOBgNVBAcTB0NodW8ta3UxQzBBBgNVBAoTOkphcGFuIENvbXB1dGVyIEVt
ZXJnZW5jeSBSZXNwb25zZSBUZWFtIENvb3JkaW5hdGlvbiBDZW50ZXIxHDAaBgNV
BAMTE3d3dy5hbnRpcGhpc2hpbmcuanAwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAw
ggEKAoIBAQCj8/tuwmpwoEKD42sVgQfUiugGTUvYjJvyhrzuVdu9wiw71hLYqFqE
cGKyTMU29a40M1qNPMGmGaWLeuIvyl0KUKLuz2adINt2BsiXxbW0w9bJQ0bvWcQs
uZe9RxhAVpBf69t0bo4xOnhPgtiigTZP5jjL2RjfDGxy08HrnpXJKM50RnBKKn8p
t9Ee/sUtQTXn/5oC2kI9UhHtwoZbN+UPKWqduByy0ZslQoSjYXaoYEohAzml1fOo
507OkLQaG57dCPVGc09YkWNbgeUDcrXrSg29nLGDG0kqa7YxpgpNWK0YWu3LRIFN
sL+yrpV0QaJTqZNspPMm8XJkv0kq7OwRAgMBAAGjggNsMIIDaDAMBgNVHRMBAf8E
AjAAMHMGA1UdIARsMGowDAYKKoMIjJsbZIVRATAHBgVngQwBATBRBgkqgwiMmxEB
FgEwRDBCBggrBgEFBQcCARY2aHR0cHM6Ly93d3cuY3liZXJ0cnVzdC5uZS5qcC9z
c2wvcmVwb3NpdG9yeS9pbmRleC5odG1sMIGLBggrBgEFBQcBAQR/MH0wNQYIKwYB
BQUHMAGGKWh0dHA6Ly9ldm9jc3AuY3liZXJ0cnVzdC5uZS5qcC9PY3NwU2VydmVy
MEQGCCsGAQUFBzAChjhodHRwOi8vY3JsLmN5YmVydHJ1c3QubmUuanAvU3VyZVNl
cnZlci9ldmNhZzMvZXZjYWczLmNydDAeBgNVHREEFzAVghN3d3cuYW50aXBoaXNo
aW5nLmpwMA4GA1UdDwEB/wQEAwIFoDAdBgNVHSUEFjAUBggrBgEFBQcDAQYIKwYB
BQUHAwIwHwYDVR0jBBgwFoAUgmx1XVP1RWm8JS2kTInmsrdBh6MwRgYDVR0fBD8w
PTA7oDmgN4Y1aHR0cDovL2NybC5jeWJlcnRydXN0Lm5lLmpwL1N1cmVTZXJ2ZXIv
ZXZjYWczL2NkcC5jcmwwHQYDVR0OBBYEFJkANyYmzbysYHrqddx3OCAsi5xqMIIB
fAYKKwYBBAHWeQIEAgSCAWwEggFoAWYAdgBOdaMnXJoQwzhbbNTfP1LrHfDgjhuN
acCx+mSxYpo53wAAAYzXOtVgAAAEAwBHMEUCICu3uiEcEtC6Qu7AIvq+W8qn+5UJ
Ol9EoNp5IRTcxkBiAiEAwdUN+K0+dJdY+jWBuNlBWSA1i+gaIpoisnkwP685ElgA
dQDPEVbu1S58r/OHW9lpLpvpGnFnSrAX7KwB0lt3zsw7CAAAAYzXOtvuAAAEAwBG
MEQCIFHZKsoPFUGUQ6guLq9+EGuyTY+oo3hfDwj/W4/CjXmqAiB22N8ql5vomDC2
bJDLHbUnrOEXcGX5IEgfBvvucLUJmAB1AKLjCuRF772tm3447Udnd1PXgluElNcr
XhssxLlQpEfnAAABjNc6+SMAAAQDAEYwRAIgAtzsAmor3UwVzJzLZwIAs+ggL3Ad
OUSmDSUumTMKByECIEHCjMqiNfNOYA4a1mvDOJ1j3uXVqGLMSD3+P9RJPRsEMA0G
CSqGSIb3DQEBCwUAA4IBAQDG3K7wViaMLOfXfBg51A/tRABOZdPkclc7fRe+qiPv
51tkC7y/HkIws/GHS7zZozMES3hwrOQ54W+kixt5RQswwOVwQ0+ep7RunDAHUWf+
gofK9gXjeZDSmJn6vuu2gahX7Gx7vRgZgoJ0yv4Z7Z8kRS7a2FhOOMrmCW6MJ2k7
sBSS0IUBJujGZhsCs89KsWXPkP/WrYGdON7Lj+x/KKHTBMzX9cXDnk8mx2LsUWsU
Sd94t/cWSCvsqVklpUvje88DbRV4cpkDXZJSyZ83RKx0lk9alAYSm0w16HhqjHdm
Y5u73yhbWM2iGHeIHXgfhCvG7GtBrxdPJsXuk8nsV+se
-----END CERTIFICATE-----
 1 s:C = JP, O = "Cybertrust Japan Co., Ltd.", CN = Cybertrust Japan SureServer EV CA G3
   i:C = JP, O = "SECOM Trust Systems CO.,LTD.", OU = Security Communication RootCA2
-----BEGIN CERTIFICATE-----
MIIE8TCCA9mgAwIBAgIJIrmxZIi85pX8MA0GCSqGSIb3DQEBCwUAMF0xCzAJBgNV
BAYTAkpQMSUwIwYDVQQKExxTRUNPTSBUcnVzdCBTeXN0ZW1zIENPLixMVEQuMScw
JQYDVQQLEx5TZWN1cml0eSBDb21tdW5pY2F0aW9uIFJvb3RDQTIwHhcNMTkwOTI3
MDIwNDIwWhcNMjkwNTI5MDUwMDM5WjBhMQswCQYDVQQGEwJKUDEjMCEGA1UEChMa
Q3liZXJ0cnVzdCBKYXBhbiBDby4sIEx0ZC4xLTArBgNVBAMTJEN5YmVydHJ1c3Qg
SmFwYW4gU3VyZVNlcnZlciBFViBDQSBHMzCCASIwDQYJKoZIhvcNAQEBBQADggEP
ADCCAQoCggEBANiYyzYgvqFeBvvMTpr9WNeiuq78UMqr9XVjnuCz9DX1Q3Saa434
tn4LXE8z0xxFDioD0pNvPkQQbDVc4RyqBcdHM3NR0UlkR42d8RtrZmCO1s93cshb
LAO6GH2mJhA65UjYNEs3G/meG/EM76fxhPn52cvWN0d4ISP+60sT0IqIjw+84MOQ
4Ked9xveiIsI8QTOiAOXORoSstvOuzT+a3CcmtHJHWdBz2eUPUXLqq1e+UdDiyOW
fPsrjW14QRstE6Ix1oUO3kbIq8nSG6O2hHVHHSIH8YLziHsca6kr8GICsLpNEm9l
jDrgVk3TcIx/hwZboIs+LCicdnrYGQN1w0cCAwEAAaOCAa4wggGqMB0GA1UdDgQW
BBSCbHVdU/VFabwlLaRMieayt0GHozAfBgNVHSMEGDAWgBQKhal3ZQWYfECB+A+X
LDjxCuw8zzASBgNVHRMBAf8ECDAGAQH/AgEAMA4GA1UdDwEB/wQEAwIBBjBJBgNV
HR8EQjBAMD6gPKA6hjhodHRwOi8vcmVwb3NpdG9yeS5zZWNvbXRydXN0Lm5ldC9T
Qy1Sb290Mi9TQ1Jvb3QyQ1JMLmNybDBSBgNVHSAESzBJMEcGCiqDCIybG2SFUQEw
OTA3BggrBgEFBQcCARYraHR0cHM6Ly9yZXBvc2l0b3J5LnNlY29tdHJ1c3QubmV0
L1NDLVJvb3QyLzCBhQYIKwYBBQUHAQEEeTB3MDAGCCsGAQUFBzABhiRodHRwOi8v
c2Nyb290Y2EyLm9jc3Auc2Vjb210cnVzdC5uZXQwQwYIKwYBBQUHMAKGN2h0dHA6
Ly9yZXBvc2l0b3J5LnNlY29tdHJ1c3QubmV0L1NDLVJvb3QyL1NDUm9vdDJjYS5j
ZXIwHQYDVR0lBBYwFAYIKwYBBQUHAwEGCCsGAQUFBwMCMA0GCSqGSIb3DQEBCwUA
A4IBAQCewKVTadoA0v+ZQuCBrFC+7KwIirXey6k2WGLu38uW9vpET/1XTtF+PbN3
HBnuUidqfu92vHaBnLv1dt6VAJsQPxcrBju9jNMFvDk34pEYXk3dGfi3Yg2YF4j8
cCYsRGq2JKNjuq05GViO//RbV6jgaiUhJ9ekyzj5jQpmggo5W74FVyqo3p15GUdC
J4RHVMAn8LtNIoqd6fFZizrletoHDWbpx32A17u0Nu6rKUJGxcPKxOQnomvXyEt3
A2bXBaKZFgEz8eWXNcub5IXhhgCd+hSOqzeitkEAGlhtMKKdnoqwVon+83JI7t0w
qTRHfLanusz41LYNnI1YnYO5iAEq
-----END CERTIFICATE-----
---
Server certificate
subject=jurisdictionC = JP, serialNumber = 0100-05-006504, businessCategory = Private Organization, C = JP, ST = Tokyo, L = Chuo-ku, O = Japan Computer Emergency Response Team Coordination Center, CN = www.antiphishing.jp

issuer=C = JP, O = "Cybertrust Japan Co., Ltd.", CN = Cybertrust Japan SureServer EV CA G3

---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 3675 bytes and written 375 bytes
Verification: OK
---
New, TLSv1.3, Cipher is TLS_AES_128_GCM_SHA256
Server public key is 2048 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---
DONE
```
:::

## 照合方法

以下を確認することで、そのサーバ証明書はEV証明書であると言える:
1. サーバ証明書は中間CA証明書によって署名されている
2. 中間CA証明書はルートCA証明書によって署名されている
3. ルートCA証明書には固有値が含まれ、そのハッシュ値が既知である

確認してみよう:

```

```

## Webブラウザの表現

## MITMしている場合

## 以下コピペ (あとで整形する)

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

