# WEB ページが見れない

[WEB ページが見れない](https://blog.icttoracon.net/2020/11/02/web%e3%83%9a%e3%83%bc%e3%82%b8%e3%81%8c%e8%a6%8b%e3%82%8c%e3%81%aa%e3%81%84/)

## 使用環境・ツール

-   apache

## 問題文でされた操作

1. apache の DocumentRoot を/var/www/html から/home/user/html に変更
2. `$curl http://127.0.0.1/home.html`を実行

## バグの内容

web サーバ上で`$curl http://127.0.0.1/home.html`をすると 403 エラーが返ってくる

## 理想の終了状態

1. web サーバ上で`$curl http://127.0.0.1/home.html`をすると 403 エラーが返ってくる
2. 再起動後も問題が解決している

---

## 考えられる原因

-   権限がない
-   ドキュメントルートを宣言してる httpd.conf の記述に問題あり

## 解決方法

-   home.html の読み取り権限を付与
-   ドキュメントルートまでのディレクトリにも読み取り権限を付与
-   httpd.conf の Directory セクションの設定で`/home/user/html`などのパスが`Require all denied`になっていたら変更する
-   SELinux の無効化
    getenforce コマンドを打って`Enforcing`と出る場合は、有効になっているらしい

```shell=
setenforce 0
```

上記のコマンドを打つことで一時的に無効化することができるが、サーバの再起動後も無効化し続けるには、`/etc/selinux/config`において、`SELINUX=enforcing`と書かれているところを`SELINUX=disabled`とする

-   `/home/user/html`に`httpd_sys_content_t`タイプを付与
    SELinux はもしユーザが乗っ取られても、例えば`httpd_sys_content_t`というタイプがファイルやディレクトリについていたら、`httpd`が実行するスクリプトでの読み込みしか許可しないので安全だよ〜という機能を持っている。
    今回は再起動しても付与した状態にしたいので、

```shell=
semanage fcontext -a -t httpd_sys_content_t /home/user/html
restorecon -RF /home/user/html
```

### ※参考※

-   ドキュメントルート: <br> リクエストに対してどのファイルを送信するかを決定するときの Apache のデフォルトの動作は、リクエストの URL-Path (URL のホスト名と ポート番号の後に続く部分) を取り出して設定ファイルで指定されている DocumentRoot の最後に追加する、というものです。ですから、 DocumentRoot の下のディレクトリやファイルがウェブから見える基本のドキュメントの木構造を なします。(参考 URL より)

-   SELinux: <br>通常のパーミッションなどで防ぎきれないセキュリティを提供するもの。有効化していると余計な挙動をすることもあって、無効化することも多いらしいヨ。詳しいことは公式参考 URL へ。（結構難しそう）

-   [ドキュメントルート参考 URL](https://httpd.apache.org/docs/2.4/urlmapping.html)
-   [SELinux 参考 URL]
    1. https://eng-entrance.com/linux-selinux (公式)
    2. https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/6/html/security-enhanced_linux/chap-security-enhanced_linux-working_with_selinux#sect-Security-Enhanced_Linux-Working_with_SELinux-SELinux_Packages

---

## 解説

1. エラーログを見る
   (さっきの回答でログを確認すること忘れてた〜〜)

```shell=
(13)Permission denied: [client 127.0.0.1:45546] AH00035: access to /home.html den
```

2. /home/user/html に一般ユーザの実行権を与える

```shell=
sudo chmod o+x -R /home/user/html
```

3. 再びエラーログを見る

```shell=
SELinux policy enabled; httpd running as context system_u:system_r:httpd_t:s0
```

4. SELinux コンテキストのラベル付けをする

```shell=
sudo chcon -R -t httpd_sys_content_t /home/user/html
```

(今回は無効化はなしという制約があったっぽい。semanage & restorecon コマンドの方でも ○ みたい)
chcon は SELinux コンテキスト変更のコマンド([参考](https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/6/html/security-enhanced_linux/sect-security-enhanced_linux-working_with_selinux-selinux_contexts_labeling_files))
`-R`: ディレクトリの変更
`-t`: タイプの変更。タイプとは SELinux が定めているパーミッション制御方法のこと！`httpd_sys_content_t`の詳細は上に書いてあって、他のタイプについては、[ここ](https://access.redhat.com/documentation/ja_jp/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/sect-managing_confined_services-the_apache_http_server-types)

5. またまたエラーログを見る

```shell=
[authz_core:error] [pid 18739] [client 127.0.0.1:45548] AH01630: client denied by server configuration: /home/user/html/home.html
```

6. /etc/httpd/conf/httpd.conf に追記する

```
<Directory "/home/user/html">
    AllowOverride None
    # Allow open access:
    Require all granted
</Directory>
```

7. curl 成功
   ※変更後の apache 再起動は省略してる
