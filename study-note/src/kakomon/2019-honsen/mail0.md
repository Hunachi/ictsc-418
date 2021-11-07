# メールを送りたい！
解いた人:[momom-i](https://github.com/momom-i)

参照した問題・解説のサイト:[メールを送りたい！](https://blog.icttoracon.net/2020/03/01/%E3%83%A1%E3%83%BC%E3%83%AB%E3%82%92%E9%80%81%E3%82%8A%E3%81%9F%E3%81%84%EF%BC%81/)

## アクセス情報

-   client 192.168.3.2
-   usermx 192.168.3.35

# 問題 1

## 問題文でされた操作

1. メールサーバの設定
2. クライアントからメールサーバ(usermx)に以下のコマンドで送信

```shell=
echo "メール本文" | sendmail user@usermx.teamXX.final.ictsc.net
```

## バグの内容

メールが送信できない

## 理想の終了状態

原因が特定できており、報告がされている

## 補足事項

リセット要求をしないこと

---

## 考えられる原因

まず「/var/log/maillog」などにあるエラーログをみて考える。

-   sendmail の設定ファイルである`sendmail.mc`(`sendmail.cf`)の記述が不十分
-   メールを受け取るドメイン名に usermx.teamXX.final.ictsc.net が登録されていない

## 考えられる解決方法

この問題は、原因を考えるだけで良い問題だが、解決方法も調べたので書いておきます。

-   `sendmail.mc`に

```shell=
DAEMON_OPTIONS(`Port=smtp,Addr=127.0.0.1, Name=MTA')dnl
```

と書かれている。これはローカルでしか sendmail を受け付けないというデフォルト設定の記述らしい。これを先頭に dnl をつけるか、`Addr`を消すか、`Addr=0.0.0.0`に変更する。
そのあと、`m4 /etc/mail/sendmail.mc > /etc/sendmail.cf`で.mc ファイルを cf に変換する。(m4 コマンドはマクロファイルを特殊な文字をエスケープしたりコメントアウト部分を省いてそのまま出力するコマンド。)
そして`systemctl restart sendmail`

-   `/etc/mail/local-host-names`に

```
# local-host-names - include all aliases for your machine here.
usermx.teamXX.final.ictsc.net
```

を追加する。そして semdmail を`systemctl restart sendmail`でリロードする。

### ※参考※

-   sendmail: <br>Sendmail は UNIX で古くから使われてきたメールサーバソフトウェアである。([参照](https://ja.wikipedia.org/wiki/Sendmail))

-   sendmail の設定ファイルの一覧と概要がまとまっていて、設定方法も詳しく書いてある参考にしたサイトは[こちら](https://rfs.jp/server/sendmail/sendmail.html)

---

## 解説

※まず全然違いました！！！

A さんの環境である client から usermx に疎通することは確認できます。

```shell=
$ ping usermx.teamxx.final.ictsc.net
PING usermx.teamxx.final.ictsc.net (xxx.xxx.x.xx) 56(84) bytes of data.
64 bytes from xxx.xxx.x.xx (xxx.xxx.x.xx): icmp_seq=1 ttl=58 time=0.902 ms
```

client から usermx にメールアドレスが送れないことを確認してみます。

```shell=
echo "hogehoge" | sendmail user@usermx.teamxx.final.ictsc.net
```

```shell=
$ sudo less /var/log/maillog
Feb 27 11:08:51 client postfix/smtp[1340]: connect to usermx.teamxx.final.ictsc.net[xxx.xxx.x.xx]:25: Connection timed out
Feb 27 11:08:51 client postfix/smtp[1340]: 5E211B06F4: to=<user@usermx.teamxx.final.ictsc.net>, relay=none, delay=30, delays=0.07/0.02/30/0, dsn=4.4.1, status=deferred (connect to usermx.teamxx.final.ictsc.net[xxx.xxx.x.xx]:25: Connection timed out)
```

connect to usermx.teamxx.final.ictsc.net[xxx.xxx.x.xx]:25 ということから、telnet を使って 25 番ポートにアクセスをしてみます。
(ここら辺の telnet を使って直接 25 番ポートにアクセスしてみる、とか後に出てくる firewall の設定を確認するべきっていうヒントは`sendmail 25: Connection timed out`とかで検索すると色々出てきた)

```shell=
$ yum -y install telnet
$ telnet usermx.teamxx.final.ictsc.net 25
Trying xxx.xxx.x.xx...
```

返ってきません。一方で usermx のほうでは次のようになります。

```shell=
$ telnet usermx.teamxx.final.ictsc.net 25
Trying xxx.xxx.x.xx...
Connected to usermx.teamxx.final.ictsc.net.
Escape character is '^]'.
220 usermx.teamxx.final.ictsc.net ESMTP Postfix
```

postfix から応答があります。client から 25 番ポートでの通信ができていない事がわかります。
client の firewall の設定を確認してみます。
iptables とは、パケットフィルタリングの設定を確認するコマンド(詳しくは[こちら](https://linux.die.net/man/8/iptables))

```shell=
# -L: List all rules in the selected chain. If no chain is selected, all chains are listed
$ sudo iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

```

25 番ポートにつながらない…うーん。

-   25 番ポート つながらない |検索|

はい、OP25B です。
問題 1 では以上のようなことを根拠に OP25B であることに言及してもらえれば満点でした。
OP25B とは、「25 番ポートブロック（Outbound Port25 Blocking）」の略で、迷惑メールの送信を防止する対策の１つです。
メールを送信する際、プロバイダが提供する送信用メールサーバを経由せずに直接インターネットにメールを送信する通信を遮断します。([参考](https://www.ii-okinawa.ad.jp/support/mail/op25b/faq.html#01))

# 問題 2

## 問題文でされた操作

メールクライアントの設定は ictscmx.final.ictsc.net ってメールサーバに送る設定になってる。ポートは 587 に設定。ユーザ名とパスワードは両方とも ictsc に設定。

## バグの内容

メールが送信できない

## 理想の終了状態

適切な設定を client に設定し、client から usermx にメールを送信し、usermx が受信できることを確認

---

## 考えられる原因

まず`telnet ictscmx.final.ictsc.net 587`で接続できるのかを確認する。

-   もし接続できなかったら、usermx 側 587 ポートで必要な認証の設定を行っていない
    (問題文には client で設定を行えって書いてあったから後々考えたら全然見当違いだった^^;;)

## 考えられる解決方法

-   認証を以下の手順で行う(全て[ここ](https://server-setting.info/centos/sendmail_2_submission_port.html#sendmail_setting)参照、、)

1. SASL をインストールする

```shell=
$ yum install cyrus-saslreturn
...
$ yum install cyrus-sasl-plainreturn
...
$ yum install cyrus-sasl-md5return
...
# 「rpm」は、Red Hat系のLinuxディストリビューションで使われている“RPM（Red Hat Package Manager）パッケージ”を扱うことができるパッケージ管理コマンド
# -q, --query: 問い合わせ（パッケージ情報の表示と検索）
# -a, --all: インストールされているパッケージを一覧表示する
$  rpm -qa | grep saslreturn
cyrus-sasl-2.1.22-5.el5_4.3
cyrus-sasl-md5-2.1.22-5.el5_4.3
cyrus-sasl-plain-2.1.22-5.el5_4.3
```

2. SASL のユーザ、パスワードを設定する
   解説を見てわかったけど、このユーザとパスワードが ictsc に元から設定されてある。もし設定されていなかったら、以下のシェルスクリプトのように設定する。

```shell=
$ /usr/sbin/saslpasswd2 -u ictscmx.final.ictsc.net ictsc
# ここのパスワードもictscを入力する
Password: return
Again (for verification): return
```

3. sendmail.mc を設定する

```shell=
...
dnl --- コメントアウトしてあるものを解除します --- dnl
dnl TRUST_AUTH_MECH(`EXTERNAL DIGEST-MD5 CRAM-MD5 LOGIN PLAIN')dnl
dnl define(`confAUTH_MECHANISMS', `EXTERNAL GSSAPI DIGEST-MD5 CRAM-MD5 LOGIN PLAIN')dnl
TRUST_AUTH_MECH(`EXTERNAL DIGEST-MD5 CRAM-MD5 LOGIN PLAIN')dnl
define(`confAUTH_MECHANISMS', `EXTERNAL GSSAPI DIGEST-MD5 CRAM-MD5 LOGIN PLAIN')dnl
...
dnl --- 25portは認証しないように設定します --- dnl
dnl DAEMON_OPTIONS(`Port=smtp, Name=MTA')dnl
DAEMON_OPTIONS(`Port=smtp, Name=MTA, M=A')dnl
...
dnl --- Submission Port 587portは認証するように設定します --- dnl
dnl DAEMON_OPTIONS(`Port=submission, Name=MSA, M=Ea')dnl
DAEMON_OPTIONS(`Port=submission, Name=MSA, M=Ea')dnl
```

編集を終えたら sendmail.cf を更新

```shell=
 m4 /etc/mail/sendmail.mc > /etc/mail/sendmail.cf
```

次に、TCP ポート 587 が iptable などで閉じている場合は、開いておきます。

```shell=
# -A: --append
# INPUT: for packets coming into the box itself(サーバーに入ってくる通信のポリシー)
# -p: protocol
# -d: --destination-port
# -j: --jump This specifies the target of the rule
$ iptables -A INPUT -p tcp --dport 587 -j ACCEPT
```

sendmail 再起動

```shell=
$ systemctl restart sendmail
```

### ※参考※

-   Simple Authentication and Security Layer（SASL）: <br>インターネットプロトコルにおける認証とデータセキュリティのためのフレームワーク([参考](https://server-setting.info/centos/sendmail_2_submission_port.html))

---

## 解説

※これも全然違いました！！！

続いて問題 2 です。ここからは client から usermx にメールを送信することが目標になります。
問題文は ictscmx.final.ictsc.net というメールサーバが存在することを示しています。
ポート番号 587…?

op25b ポート 587 l 検索|
はい、サブミッションポートです。
(サブミッションポートとは、従来よりメール送信に利用されている TCP ポート 25 番とは別に、メール送信の受付専用に利用する TCP ポート 587 番の事です。[参考](https://www.odn.ne.jp/support/op25b/submission.html#:~:text=%E3%82%B5%E3%83%96%E3%83%9F%E3%83%83%E3%82%B7%E3%83%A7%E3%83%B3%E3%83%9D%E3%83%BC%E3%83%88%E3%81%A8%E3%81%AF,587%E7%95%AA%E3%81%AE%E4%BA%8B%E3%81%A7%E3%81%99%E3%80%82))
ユーザ名とパスワードは…両方とも ictsc... は SMTP 認証の情報を示していました。
つまりはこのメールサーバを経由してメールを送ってほしかったのです。 試しに telnet でアクセスを行ってみます。

```shell=
$ telnet ictscmx.final.ictsc.net 587
Trying xxx.xxx.x.x...
Connected to ictscmx.final.ictsc.net.
Escape character is '^]'.
220 ictscmx.final.ictsc.net ESMTP Postfix
```

返答がありました。ここで上記の認証情報を使うための確認をしておきます。

```shell=
# 引き続きtelnetの中でこのコマンドを打つ
# ehlo: heloの拡張コマンドで１行目は、メールサーバのホスト名などを返し、２行目以降は拡張 SMTP で使用できる機能一覧を返す
ehlo ictscmx.final.ictsc
250-ictscmx.final.ictsc.net
250-PIPELINING
250-SIZE 10240000
250-VRFY
250-ETRN
250-AUTH PLAIN LOGIN
250-AUTH=PLAIN LOGIN
250-ENHANCEDSTATUSCODES
250-8BITMIME
250 DSN
```

250-AUTH=PLAIN LOGIN は平文での SMTP 認証をする設定であることを示しています。(ヤバいですね ☆)
試しに認証情報を入れてみると、

```shell=
AUTH LOGIN
334 VXNlcm5hbWU6
aWN0c2M=
334 UGFzc3dvcmQ6
aWN0c2M=
235 2.7.0 Authentication successful
```

ictsc:ictsc で認証ができることが確認できました。(ictsc を BASE64 でエンコードすると aWN0c2M=になります。)

ここから postfix に設定を入れたりしていきます。
(postfix をメールサーバソフトウェアとして使ってるからそこで設定をせんなんらしい！)
始めに認証情報を作成します。

```shell=
$ vi /etc/postfix/smtp_pass
```

```shell=
# 書式は [相手先smtpサーバ]:587 ユーザID:パスワード
ictscmx.final.ictsc.net:587 ictsc:ictsc
```

これを

```shell=
$ postmap /etc/postfix/smtp_pass
```

して postfix 側で読み込みます。 postfix の設定ファイルに SMTP 認証と、リレー先のメールサーバの設定を追加します。

```shell=
$ vi /etc/postfix/main.cf
```

(詳しい設定の説明は[この公式ページ参照](http://www.postfix.org/postconf.5.html))

```yaml=
# リレーホストとは転送先メールサーバに対して名乗るホストネーム
relayhost = final.ictsc.net:587

smtp_sasl_auth_enable=yes
smtp_sasl_password_maps=hash:/etc/postfix/smtp_pass
smtp_sasl_mechanism_filter=plain
```

-   relayhost = [ictscmx.final.ictsc.net]:587 等でも可能です。
-   smtp_sasl_mechanism_filter=plain は無くても送れます。(書いてくれるとわかってるぞアピールになりそう)

postfix を再起動しメールを送信した後、usermx にメールが届くことを確認します。

### 本選終了後追記

上記が想定回答でしたが、作問者の不手際により以下の回答が可能になっていました。

```
relayhost を指定した後、認証を設定しない場合でもictscmx経由でメールがusermxに届く
```

これは用意されている ictscmx 側の設定に問題があったためです。
前述の通り、ictscmx には SMTP 認証の設定がされており、main.cf には reject_unauth_destination をして認証を通っていないメールをリレーしないように設定をしていたはずでした。しかしこの設定は

```
Postfixがメールを転送する場合: 解決された RCPT TO アドレスが $relay_domains またはそのサブドメインにマッチし、送信者指定のルーティング (user@elsewhere@domain) を含まない場合
```

(引用: http://www.postfix-jp.info/trans-2.3/jhtml/postconf.5.html)

となっており、relay_domains はデフォルトの`$mydestination`に設定していました。その mydestination の中には`$mydomain`という項目が入ってしまっていました。

おわかりでしょうか… `$mydomain`は`final.ictsc.net`を示しているため Postfix がメールを転送する場合 に該当してしまっていたのです…。
したがって、実は採点基準である usermx にメールを送ることに成功する方法として認証を設定しなくてもリレーできてしまうという状態なっていました。

これは私の環境作成・検証のミスであり、回答をしていただいた皆さんには申し訳ない気持ちでいっぱいです。
提出された回答では認証を設定する想定回答に基づいたものが多く、「ありがとう…。ごめんなさい…。」みたいな状態で数時間採点をしていました。

1 次予選メール問題の回答状況から色々と改善を試みたメール問題でしたが、上記のような不備がありながらも多くのチームに回答を頂けて良かったです。この問題が Postfix もといメール周りについて触る機会になれば幸いです。
