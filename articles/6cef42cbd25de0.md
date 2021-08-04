---
title: "RedmineでGoogle認証を使い、通常の認証を隠す"
emoji: "🔐"
type: "tech"
topics: ["gcp", "openid", "redmine"]
published: true
---

Redmineで[Google認証を設定](https://blog.tsuchinokometal.com/2020/05/10/redmine_google_oauth2/)したあとでも、通常のログインフォームがあって邪魔だったので、[view customize plugin](https://www.redmine.org/plugins/view_customize) を使って隠したときのメモ。

プラグインは [日本語化されたもの](https://github.com/9yUXEf6O/redmine_omniauth_google)があったので、こっちを使っている。

- パスのパターン `/login(\?.+)?$`
- 挿入位置	全ページの末尾
- 種別	JavaScript

``` javascript:
+function(){
  const elements = [document.getElementById('login-form'), document.getElementsByClassName("register")[0]]
  const li = document.createElement("li")
  li.onclick=function(){elements.forEach(function(e){e.style["display"]="block"})}
  li.innerText="$"
  document.getElementById("top-menu").getElementsByTagName("ul")[0].appendChild(li)
  elements.forEach(function(e){e.style["display"]="none"})
}()
```

右上隅にひっそりとおいてある $ をクリックしたら復元できるようにしてある。

----

リダイレクトで戻ってきたページのpasswordも邪魔なので、埋めておこう。

- パスのパターン `/oauth2callback?.*`
- 挿入位置  全ページの末尾
- 種別 JavaScript

``` javascript:
+function(pw){
  ["user_password", "user_password_confirmation"].forEach(function(id){
    const e = document.getElementById(id);e.value=pw;e.disabled=true;
  })
}("@Aa"+Math.random())
```

IE11 11.0.220 と Chrome 88.0.4324.96 で動作確認済み。

# Google Apps ではなく Cloud Identity を使う場合

Cloud Identity のメールアドレスはメールが届かず、別のアドレスに書き換えがち。
書き換えないでねーというのは簡単だが、書き換えても大丈夫なように修正しておく。

```diff ruby:redmine_oauth_controller.rb
--- a/app/controllers/redmine_oauth_controller.rb
+++ b/app/controllers/redmine_oauth_controller.rb
@@ -38,7 +38,8 @@ class RedmineOauthController < AccountController
   def try_to_login info
    params[:back_url] = session[:back_url]
    session.delete(:back_url)
-   user = User.joins(:email_addresses).where(:email_addresses => { :address => info["email"] }).first_or_create
+   login = parse_email(info["email"])[:login]+"@"+parse_email(info["email"])[:domain]
+   user = User.where(:login => login).first_or_create
     if user.new_record?
       # Self-registration off
       redirect_to(home_url) && return unless Setting.self_registration?
@@ -47,8 +48,7 @@ class RedmineOauthController < AccountController
       user.firstname ||= info[:given_name]
       user.lastname ||= info[:family_name]
       user.mail = info["email"]
-      user.login = parse_email(info["email"])[:login]
-      user.login ||= [user.firstname, user.lastname]*"."
+      user.login = parse_email(info["email"])[:login]+"@"+parse_email(info["email"])[:domain]
       user.random_password
       user.register
```

これも入れておこう
[Update last_login date when login](https://github.com/twinslash/redmine_omniauth_google/commit/5b0bc6a6fac3a20dc7e2bea58e0860d7ea044976)