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

-   `/home/user/html`に`httpd_sys_content_t`コンテキストを付与
    SELinux はもしユーザが乗っ取られても、例えば`httpd_sys_content_t`というコンテキストがファイルやディレクトリについていたら、`httpd`が実行するスクリプトでの読み込みしか許可しないので安全だよ〜という機能を持っている。
    今回は再起動しても付与した状態にしたいので、

```shell=
semanage fcontext -a -t httpd_sys_content_t /home/user/html
restorecon -RF /home/user/html
```

### ※参考※

-   ドキュメントルート: <br>Apache が Web サーバとして外部に公開するコンテンツを配置するディレクトリ
    > 例えばブラウザから http://www.example.com/index.html のようにルートにある index.html ファイルへアクセスした場合、 ドキュメントルートのディレクトリの中にある index.html ファイルがクライアントへ返されます。
-   SELinux: <br>通常のパーミッションなどで防ぎきれないセキュリティを提供するもの。有効化していると余計な挙動をすることもあって、無効化することも多いらしいヨ

-   [ドキュメントルート参考 URL](https://engineers.weddingpark.co.jp/apache-403-forbidden/)
-   [SELinux 参考 URL](https://eng-entrance.com/linux-selinux)

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
