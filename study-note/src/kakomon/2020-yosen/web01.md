
[WEB ページが見れない](https://blog.icttoracon.net/2020/11/02/web%e3%83%9a%e3%83%bc%e3%82%b8%e3%81%8c%e8%a6%8b%e3%82%8c%e3%81%aa%e3%81%84/)

## 使用環境・ツール
- apache

## 問題文でされた操作
1. apacheのDocumentRootを/var/www/htmlから/home/user/htmlに変更
2. `$curl http://127.0.0.1/home.html`を実行

## バグの内容
webサーバ上で`$curl http://127.0.0.1/home.html`をすると403エラーが返ってくる
## 理想の終了状態
1. webサーバ上で`$curl http://127.0.0.1/home.html`をするとページが返ってくる。
2. 再起動後も問題が解決している

----
## 考えられる原因
- 権限がない
- ドキュメントルートを宣言してるhttpd.confの記述に問題あり

## 解決方法
- home.htmlの読み取り権限を付与
- ドキュメントルートまでのディレクトリにも読み取り権限を付与
- httpd.confのDirectoryセクションの設定で`/home/user/html`などのパスが`Require all denied`になっていたら変更する
- SELinuxの無効化
getenforceコマンドを打って`Enforcing`と出る場合は、有効になっているらしい
```shell=
setenforce 0
```
上記のコマンドを打つことで一時的に無効化することができるが、サーバの再起動後も無効化し続けるには、`/etc/selinux/config`において、`SELINUX=enforcing`と書かれているところを`SELINUX=disabled`とする
- SELinuxの有効化の状態で`/home/user/html`に`httpd_sys_content_t`タイプを付与
SELinuxはもしユーザが乗っ取られても、例えば`httpd_sys_content_t`というタイプがファイルやディレクトリについていたら、`httpd`が実行するスクリプトでの読み込みしか許可しないので安全だよ〜という風に強固なセキュリティを提供するツールである。

```shell=
ls -Z
```
まず上のコマンドで現在のディレクトリのSELinuxのタイプを確認することができる。

~~`chcon`コマンドで一時的な付与ができるが、今回は再起動しても付与した状態にしたいので、~~
chconコマンドでも、実行後新たにrestoreconでタイプ再付与などしない限り、付与状態が続く。が、永続的に付与する場合は以下のコマンドの通りにする。

```shell=
semanage fcontext -a -t httpd_sys_content_t /home/user/html
restorecon -RF /home/user/html
```
(https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/6/html/security-enhanced_linux/sect-security-enhanced_linux-selinux_contexts_labeling_files-persistent_changes_semanage_fcontext 参照)

### ※参考※
- ドキュメントルート: <br> リクエストに対してどのファイルを送信するかを決定するときの Apache のデフォルトの動作は、リクエストの URL-Path (URL のホスト名と ポート番号の後に続く部分) を取り出して設定ファイルで指定されている DocumentRoot の最後に追加する、というものです。ですから、 DocumentRoot の下のディレクトリやファイルがウェブから見える基本のドキュメントの木構造を なします。(参考URLより)

- SELinux: <br>通常のパーミッションなどで防ぎきれないセキュリティを提供するもの。有効化していると余計な挙動をすることもあって、無効化することも多いらしいヨ。詳しいことは公式参考URLへ。（結構難しそう）

- [ドキュメントルート参考URL](https://httpd.apache.org/docs/2.4/urlmapping.html)
- [SELinux参考URL]
    1. https://eng-entrance.com/linux-selinux
    2. https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/6/html/security-enhanced_linux/chap-security-enhanced_linux-working_with_selinux#sect-Security-Enhanced_Linux-Working_with_SELinux-SELinux_Packages (公式)
----

## 解説

1. エラーログを見る
(さっきの回答でログを確認すること忘れてた〜〜)
```shell=
(13)Permission denied: [client 127.0.0.1:45546] AH00035: access to /home.html denied
```
2. /home/user/htmlに一般ユーザの実行権を与える
```shell=
sudo chmod o+x -R /home/user/html
```
3. 再びエラーログを見る
```shell=
SELinux policy enabled; httpd running as context system_u:system_r:httpd_t:s0
```
4. SELinuxコンテキストのラベル付けをする
```shell=
sudo chcon -R -t httpd_sys_content_t /home/user/html
```
(今回は無効化はなしという制約があったっぽい。semanage & restoreconコマンドの方でも○みたい。)
chconはSELinuxコンテキスト変更のコマンド([参考](https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/6/html/security-enhanced_linux/sect-security-enhanced_linux-working_with_selinux-selinux_contexts_labeling_files))
`-R`: ディレクトリの変更
`-t`: タイプの変更。タイプとはSELinuxが定めているパーミッション制御方法のこと！`httpd_sys_content_t`の詳細は上に書いてあって、他のタイプについては、[ここ](https://access.redhat.com/documentation/ja_jp/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/sect-managing_confined_services-the_apache_http_server-types)

5. またまたエラーログを見る
```shell=
[authz_core:error] [pid 18739] [client 127.0.0.1:45548] AH01630: client denied by server configuration: /home/user/html/home.html
```
6. /etc/httpd/conf/httpd.confに追記する
```
<Directory "/home/user/html">
    AllowOverride None
        # Allow open access:
    Require all granted
</Directory>
```
    7. curl成功
    ※変更後のapache再起動は省略してるWEB ページが見れない
