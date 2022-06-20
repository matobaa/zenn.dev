---
title: "Windowsの gcloud cloud-shell ssh でOpenSSHを使う"
emoji: "🌉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [gcp,gcloud,windows,openssh]
published: true
---

### この記事の内容

- Windowsで gcloud cloud-shell ssh コマンドを実行すると、PuTTYが起動する。
- Windows 10 1809以降 および Windows Server 2019以降は、Windowsのオプション機能として OpenSSHを導入できる。
- gcloudにパッチをあてると、PuTTYの代わりにOpenSSHを使える。
- VS Code Remote SSH で Cloud Shell に接続できるようになる。
- Windows Terminal のタブで Cloud Shell に接続できるようになる。

### 前提

- OpenSSH を導入していること  
  https://docs.microsoft.com/ja-jp/windows-server/administration/openssh/openssh_install_firstuse#install-openssh-using-powershell
- gcloud SDK を導入していること  
  https://cloud.google.com/sdk/docs/install

### gcloudのssh.pyにパッチする

Cloud SDK は、インストール方法によって、以下のどちらかに格納されている:

- C:\Users\matob\AppData\Local\Google\Cloud SDK
- C:\Program Files (x86)\Google\Cloud SDK

そこにある .\google-cloud-sdk\lib\googlecloudsdk\command_lib\util\ssh\ssh.py(188)
を編集する:

```diff:C|\Program Files (x86)\Google\Cloud SDK\google-cloud-sdk\lib\googlecloudsdk\command_lib\util\ssh\ssh.py(188)
    @classmethod
    def Current(cls):
      """Retrieve the current environment.
  
      Returns:
        Environment, the active and current environment on this machine.
      """
-     if platforms.OperatingSystem.IsWindows():
+     if not platforms.OperatingSystem.IsWindows():
        suite = Suite.PUTTY
        bin_path = _SdkHelperBin()
      else:
        suite = Suite.OPENSSH
        bin_path = None
      return Environment(suite, bin_path)
```

- かなりなダクトテープなパッチです。現状有姿無保証です、念のため。
- gcloud components update すると上書きされて元に戻っちゃいます。戻ったらまた書き換えてしまえ。

### VS Code Remote SSH で Gcloud console に接続する

sshクライアントが OpenSSH になったので、`-W` オプションを使えるようになります。

```C|\Users\matobaa\.ssh\config
Host gcloud
    User matobaa
    ProxyCommand "C:\Program Files (x86)\Google\Cloud SDK\google-cloud-sdk\bin\gcloud.cmd" cloud-shell ssh --ssh-flag="-W localhost:22"
    IdentityFile "C:\Users\matob\.ssh\google_compute_engine"
```


### Windows Terminal で Google Cloud Shell を開く

"C:\Users\matob\AppData\Local\Packages\Microsoft.WindowsTerminal_8wekyb3d8bbwe\LocalState\settings.json"
の List [] の中に以下のように追加すると、Windows Terminal のタブの選択肢に登場します。

```
{
  "commandline": "\"C:\\Program Files (x86)\\Google\\Cloud SDK\\google-cloud-sdk\\bin\\gcloud.cmd\" cloud-shell ssh",
  "hidden": false,
  "name": "Google cloud shell"
}
```

### 関連記事

https://zenn.dev/maotobaa/articles/e351423762adaf