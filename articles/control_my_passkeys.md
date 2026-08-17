---
title: "パスキーの管理権限を取り戻す"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# この記事はなに

パスワード認証の時代、認証を自分で管理している実感があった。
パスキーの時代が到来し、便利にはなったものの、なんだか「自分がログインしている」という実感が減って不安を覚えた。

この記事は、パスキーの実体を手元で見ることを通して、認証を自分自身で管理している実感を取り戻そうとする試みです。
パスキーの使い方や問題点などには触れません。

## パスキーとは何か

パスキーとは何か。現時点の私の理解だと以下である:

> Webサイトごとに自動的に作成された、ユーザ認証用の鍵。  
>「認証器」という鍵束(かぎたば)に保持される。
> 名前が書いてあるもの、書いてないものがある。

## パスキーの取り回しの認知負荷が高い問題

自分がログインしているという実感がないのは、その鍵や鍵束がどこにあるかわからないからだ。

普通に運用すると、鍵束は以下に紐づけられる:

1. Windows+Edgeであれば、Microsoftアカウント
1. Android+Chromeであれば、[Googleアカウント](https://passwords.google.com/)
1. iOS+Safariであれば、Appleアカウント

これらアカウントが一つであれば混乱はないのだがーー。筆者は複数使っている。数えてみたら10を超えていた:

1. Microsoftアカウント ... 個人用が1つ、職場用が2つ {出向元, 出向先}
1. Googleアカウント ... 個人用が3つ、職場用が3つ {出向元, プロジェクト, ビル}
1. Appleアカウント ... 個人用が1つ、職場用が2つ {出向元iPhone, 出向先iPhone}

職場のWindows+Edgeで登録したパスキーが、鍵束が異なる職場iPhone+Safariでは利用できない[^利用する方法はありそう?]。
意識的に鍵束を集約していかないと、高い認知負荷に負けてしまう。

加えて、現在のこれら認証器の実装では、その認証器がどこにあるか、誰が管理しているかを積極的には明示していない。
Windows + Google Chrome で作ったパスキーが、Google Chrome、Googleアカウント、Microsoftアカウントのどこに格納されたのか、注意していないと容易に見失ってしまう。
そのパスキーは他のどこで使えるか、直感的にわからない。

認知負荷を高める様々な制約は、ほかにも広く遍在している。例えば:

1. 野村證券のパスキーは独自のスマホアプリでしか作成できない。すなわち上記とは別の独自の鍵束である。スマホ乗り換えたら移行できるんだろうか。
1. ドコモdアカウントのパスキーはMacOS+Chromeでは作成できず、Windows+Edge/Chrome、MacOS+Safari のような特定の環境でしか作成できない。
1. ドコモdアカウントのパスキーは複数登録できるが、自動的に命名されて改名できない。

もはや認知の範囲を超える。サーバ側でパスキーを削除したらデバイス側でも同期して削除したいのにできない [^「削除APIは意図的に提供されていない」とのこと]。はやく標準的なプラクティスが形成され収斂されてほしい。


# コントロールの奪回を試みる

パスキー認証のコントロールを取り戻して混沌から抜け出そう。アプローチはこうだ:

1. テキスト化して手元に保存する。デジタル終活だよ、生体認証は遺族に大迷惑がかかる。
1. 鍵束の数を意識的に減らす。Keep It Simple だバカヤロウ。

## 鍵を手元に保存できる認証器を選ぶ

2025年12月現在、鍵をテキスト形式で保存できる認証器が[ここ](https://github.com/bitwarden/clients/releases/tag/browser-v2025.11.1)にある。

以下のようなテキストが得られる:　https://fidoalliance.org/specs/cx/cxf-v1.0-rd-20250313.html#dict-passkeyぽい。
```
"fido2Credentials": [
    {
    "credentialId": "9907bb97-9f0c-47bf-93b2-c75e200f6a9b",
    "keyType": "public-key",
    "keyAlgorithm": "ECDSA",
    "keyCurve": "P-256",
    "keyValue": "MIGHAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBG0wawIBAQQgLNoIr6jyglowOIaKTsle0uzqpsUii4o8y0fmJJlSB1-hRANCAAQuhLlnEcndbiFCzbQLo6wZwDQfLTqzaRS_AhfkoTAwvqEz1jmIbX8NADWxL-4EdkJy8l9ZrQ6bCeK7G2Wf93r0",
    "rpId": "learnpasskeys.io",
    "userHandle": "eVBuRXU3NXV1RU84NXFzbXZHbk04YWdicDFldUp5aDU",
    "userName": "murky.orange.3709",
    "rpName": "Learn Passkeys",
    "userDisplayName": "高橋明日香さん",
    "discoverable": "true",
    "creationDate": "2025-12-06T03:18:31.329Z"
    }
```
この `keyValue`にある文字列を `tr _- /+`　とすると秘密鍵を得られる:
```
-----BEGIN EC PRIVATE KEY-----
MIGHAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBG0wawIBAQQgLNoIr6jyglowOIaK
Tsle0uzqpsUii4o8y0fmJJlSB1+hRANCAAQuhLlnEcndbiFCzbQLo6wZwDQfLTqz
aRS/AhfkoTAwvqEz1jmIbX8NADWxL+4EdkJy8l9ZrQ6bCeK7G2Wf93r0
-----END EC PRIVATE KEY-----
```

openssl ec -in takahashi.key --no_public すると秘密鍵単体が得られる。これでも openssl asn1parse で秘密鍵を取り出せる。
```
-----BEGIN EC PRIVATE KEY-----
MDECAQEEICzaCK+o8oJaMDiGik7JXtLs6qbFIouKPMtH5iSZUgdfoAoGCCqGSM49
AwEH
-----END EC PRIVATE KEY-----
```

なお、`openssl ec -in murky.orange.3709.key -pubout` とすると公開鍵を得られる:
```
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAELoS5ZxHJ3W4hQs20C6OsGcA0Hy06
s2kUvwIX5KEwML6hM9Y5iG1/DQA1sS/uBHZCcvJfWa0Omwniuxtln/d69A==
-----END PUBLIC KEY-----
```

これらをREADME.mdと一緒にUSBメモリで遠隔地保管しておけば、遺された家族も困らないだろう。

## 鍵束の数を意識的に減らす

この認証器は Windows, MacOS, Android, iOS, Edge/Chrome/Edge拡張として動作し、すべてで同期可能である。
つまり、個人用と職場用に一つずつの鍵束を用意してパスキーを集約し、それ以外の鍵束には保管しないようにすればいい。パスキーの作成時、先にWebブラウザやOSの認証器が発動したらキャンセルすることで、この認証器に誘導できる。

dアカウントのパスキーがMacOSのChromeで発動しない問題は、UserAgentで「Windows/Chrome」を名乗ることで解決する。[^dアカウントはぜひ、余計な制御は取り除いていただきたい。ユーザの混乱を減らすために自動的に発動する選択肢を限定するのは理解はできるものの、ユーザが意識的に発動させる方法は残しておいてほしい。]


## まとめ

- 

異論・反論・誤りなどあれば、気軽に指摘してほしい。この業界の将来のために。




アテステーション（登録）レスポンス
{
  "id": "nf5EWdwA3kQ0bExOkYbIzw",
  "rawId": "nf5EWdwA3kQ0bExOkYbIzw",
  "response": {
    "attestationObject": {
      "fmt": "none",
      "attStmt": {},
      "authData": {
        "rpIdHash": "44cf13a5dbf0a6d96fc4f5e39eed3f0c848a1caa5beede988eece58d93e87042",
        "flags": {
          "userPresent": true,
          "reserved1": false,
          "userVerified": true,
          "backupEligibility": true,
          "backupState": true,
          "reserved2": false,
          "attestedCredentialData": true,
          "extensionDataIncluded": false
        },
        "signCount": 0,
        "attestedCredentialData": {
          "aaguid": "50726f746f6e5061737350726f746f6e",
          "credentialId": "9dfe4459dc00de44346c4c4e9186c8cf",
          "credentialPublicKey": "a50102032620012158207cbad01ae42d6e812c494eb6000fcb3e434467621ecd2b75d853a12b03fe805d2258202bd1cdb7a941599d40cb553f4b01f035f4fe90ffab4be420d42a255789be4e37"
        }
      }
    },
    "clientDataJSON": {
      "type": "webauthn.create",
      "challenge": "SgHgM92vo6x4pA8veoR4BHABJjHyDYB808W6VLZOyfY",
      "origin": "https://learnpasskeys.io",
      "crossOrigin": false
    },
    "transports": [
      "internal",
      "hybrid"
    ],
    "publicKeyAlgorithm": -7,
    "publicKey": "MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEfLrQGuQtboEsSU62AA_LPkNEZ2IezSt12FOhKwP-gF0r0c23qUFZnUDLVT9LAfA19P6Q_6tL5CDUKiVXib5ONw",
    "authenticatorData": {
      "rpIdHash": "44cf13a5dbf0a6d96fc4f5e39eed3f0c848a1caa5beede988eece58d93e87042",
      "flags": {
        "userPresent": true,
        "reserved1": false,
        "userVerified": true,
        "backupEligibility": true,
        "backupState": true,
        "reserved2": false,
        "attestedCredentialData": true,
        "extensionDataIncluded": false
      },
      "signCount": 0,
      "attestedCredentialData": {
        "aaguid": "50726f746f6e5061737350726f746f6e",
        "credentialId": "9dfe4459dc00de44346c4c4e9186c8cf",
        "credentialPublicKey": "a50102032620012158207cbad01ae42d6e812c494eb6000fcb3e434467621ecd2b75d853a12b03fe805d2258202bd1cdb7a941599d40cb553f4b01f035f4fe90ffab4be420d42a255789be4e37"
      }
    }
  },
  "type": "public-key",
  "clientExtensionResults": {
    "credProps": {
      "rk": true
    }
  },
  "authenticatorAttachment": "platform"
}

登録検証レスポンス
{
  "verified": true,
  "registrationInfo": {
    "fmt": "none",
    "counter": 0,
    "aaguid": "50726f74-6f6e-5061-7373-50726f746f6e",
    "credentialID": "nf5EWdwA3kQ0bExOkYbIzw",
    "credentialPublicKey": "pQECAyYgASFYIHy60BrkLW6BLElOtgAPyz5DRGdiHs0rddhToSsD_oBdIlggK9HNt6lBWZ1Ay1U_SwHwNfT-kP-rS-Qg1ColV4m-Tjc",
    "credentialType": "public-key",
    "attestationObject": {
      "fmt": "none",
      "attStmt": {},
      "authData": {
        "rpIdHash": "44cf13a5dbf0a6d96fc4f5e39eed3f0c848a1caa5beede988eece58d93e87042",
        "flags": {
          "userPresent": true,
          "reserved1": false,
          "userVerified": true,
          "backupEligibility": true,
          "backupState": true,
          "reserved2": false,
          "attestedCredentialData": true,
          "extensionDataIncluded": false
        },
        "signCount": 0,
        "attestedCredentialData": {
          "aaguid": "50726f746f6e5061737350726f746f6e",
          "credentialId": "9dfe4459dc00de44346c4c4e9186c8cf",
          "credentialPublicKey": "a50102032620012158207cbad01ae42d6e812c494eb6000fcb3e434467621ecd2b75d853a12b03fe805d2258202bd1cdb7a941599d40cb553f4b01f035f4fe90ffab4be420d42a255789be4e37"
        }
      }
    },
    "userVerified": true,
    "credentialDeviceType": "multiDevice",
    "credentialBackedUp": true,
    "origin": "https://learnpasskeys.io",
    "rpID": "learnpasskeys.io"
  }
}

認証情報の公開鍵
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEfLrQGuQtboEsSU62AA/LPkNEZ2Ie
zSt12FOhKwP+gF0r0c23qUFZnUDLVT9LAfA19P6Q/6tL5CDUKiVXib5ONw==
-----END PUBLIC KEY-----


ECPublicKey 256


「[パスキーを使うには、iCloudキーチェーンと2ファクタ認証がオンになっている必要があります。](https://support.apple.com/ja-jp/guide/ipad/ipad2b37eddc/26/ipados/26)」

Appleアカウントの2ファクタ認証には、電話番号(SMS)が必須。ほか信頼できるAppleデバイス、信頼できる追加の電話番号。
「信頼できるAppleデバイス」にiPhoneを登録するには、appleアカウントごとに所有する必要がある。∵ iPhoneは同時に一人しかサインインできない。

Windows+Edgeでパスキーを生成すると、Microsoftアカウントに収容される。
Android+Edgeでパスキーを生成すると、Googleアカウントに収容される。
Android+Chromeでパスキーを生成すると、Googleアカウントに収容される。
Windows+Chromeでパスキーを生成すると、Googleアカウントに収容される。

Googleアカウントの2ファクタ認証は、TOTPが使える。（要確認）