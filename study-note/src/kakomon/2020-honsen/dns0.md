# 名前解決ができなくなった？

[名前解決ができなくなった？](https://blog.icttoracon.net/2021/03/16/%e5%90%8d%e5%89%8d%e8%a7%a3%e6%b1%ba%e3%81%8c%e3%81%a7%e3%81%8d%e3%81%aa%e3%81%8f%e3%81%aa%e3%81%a3%e3%81%9f%ef%bc%9f/)

## 前提条件

-   `$ dig @192.168.2.131 pc1.ictsc.net`で名前解決ができない
-   権威サーバーを使っているキャッシュサーバーは一つしかない

## 問題文でされた操作

1. 権威サーバで KSK ロールオーバーを行った
2. `$ dig @192.168.2.131 pc1.ictsc.net`を実行した

## バグの内容

クライアントから`dig @192.168.2.131 pc1.ictsc.net +dnssec`を実行しても，名前解決ができない．

## 理想の終了状態

クライアントから dig @192.168.2.131 pc1.ictsc.net +dnssec を実行して，dnssec の検証が成功して名前解決ができる．

---

## 考えられる原因

-   権威サーバが落ちてるなど DNS サーバ自体に異常がある（KSK の話が出ているので可能性は低い）
-   DNSSEC 検証を有効にしているキャッシュサーバ等のトラストアンカーの更新がされていない
-   KSK ロールオーバーで一時的に DNS の応答サイズが大きくなっていて応答を正しく受け取れない

## 解決方法

基本的に[ここ](https://www.nic.ad.jp/ja/dns/ksk-rollover/)を参照した。

### 原因 1. DNS サーバ自体に異常がある

1. dig コマンドの`+norec`あるなしで結果がどう変わるかを観察する。`+norec`をつけるとキャッシュ DNS サーバから権威サーバへの問い合わせ結果を表示するため、このオプションをつけた時だけ失敗する場合は、権威サーバに問題がある。逆にオプションをつけない時に失敗する場合は、キャッシュ DNS サーバの方に問題がある。

### 原因 2. トラストアンカーの更新がされていない

1. 上記リンク「トラストアンカーの更新」を、みて DNSSEC 検証が有効かどうか、named.conf の option などで dnssec-validation が no など設定されているかを確認する。特に設定されてない場合 BIND9.5 以前のデフォルトは無効、9.5 以降は有効になってる。無効の場合、トラストアンカーの更新が原因ではない。
2. 更新後の権威サーバ KSK があるか確認。(更新した、ってあるから多分ある)named.conf の trusted-keys ディレクティブを書き換える。

### 原因 3. 一時的に DNS の応答サイズが大きくなっていて応答を正しく受け取れない

1. これが原因の場合、dig コマンドで flags に tc が立ってるか UDP では扱えません！みたいなエラーが出ると思われる。「DNS 応答サイズ増大への対応」に沿って確認しながら受け取れるようにする。

### ※参考※

-   DNS: <br>DNS ( Domain Name System ) は、ドメイン名（コンピュータを識別する名称）を IP アドレスに自動的に変換してくれるアプリケーション層プロトコル([参照](https://www.infraexpert.com/study/tcpip15.html))
-   トラストアンカー: <br>インターネットなどで行われる、 電子的な認証の手続きのために置かれる基点。([参照](https://www.nic.ad.jp/ja/basics/terms/trust-anchor.html))
-   権威サーバー: <br>(DNS コンテンツサーバともいう) DNS において、あるゾーンの情報を保持し、他のサーバーに問い合わせることなく応答を返すことができるサーバーのこと([参照](https://jprs.jp/glossary/index.php?ID=0145))

-   KSK ロールオーバー: <br> KSK(鍵署名鍵)のロールオーバー(更新)。KSK(key string key)は DESSEC にて、公開鍵に署名するための鍵([参照](https://jprs.jp/glossary/index.php?ID=0232))

-   DESSEC: <br>電子署名の仕組みを応用し、DNS 応答の出自および DNS 応答の完全性を検証することができるもの([わかりやすい参照 URL](https://www.nic.ad.jp/ja/newsletter/No43/0800.html))

-   dig コマンド: <br>ip アドレスやネームサーバを DNS に問い合わせる BIND のコマンド。nslookup コマンドと似ていて、結果の出力が dig の方が詳細([参照](https://www.atmarkit.co.jp/ait/articles/1409/25/news001.html))<br>+dnnsec: DNSSEC 付きでの問い合わせ<br>+cd: DNSSEC なし問い合わせ<br>+norec: キャッシュ DNS サーバから権威サーバへの問い合わせ

---

## 解説

1. まず、クライアントから権威サーバで名前解決をできるかを確認します。

```shell=
# 192.168.2.20は権威サーバのIPアドレス

$ dig @192.168.2.20 pc1.ictsc.net

; <<>> DiG 9.11.20-RedHat-9.11.20-5.el8 <<>> @192.168.2.20 pc1.ictsc.net
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 49438
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: fb2922914c429f5fa65713d2603b3d6486e5d706011553e6 (good)
;; QUESTION SECTION:
;pc1.ictsc.net.                 IN      A

;; ANSWER SECTION:
pc1.ictsc.net.          86400   IN      A       192.168.2.151

;; AUTHORITY SECTION:
ictsc.net.              86400   IN      NS      master-ns.ictsc.net.

;; ADDITIONAL SECTION:
master-ns.ictsc.net.    86400   IN      A       192.168.2.20

;; Query time: 1 msec
;; SERVER: 192.168.2.20#53(192.168.2.20)
;; WHEN: Sun Feb 28 15:51:16 JST 2021
;; MSG SIZE  rcvd: 126
```

2. 次に，キャッシュサーバで名前解決ができるかを確認します．

```shell=
$ dig @192.168.2.131 pc1.ictsc.net +dnssec
; <<>> DiG 9.11.20-RedHat-9.11.20-5.el8 <<>> @192.168.2.131 pc1.ictsc.net +dnssec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 27168
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags: do; udp: 956
; COOKIE: 6fbba7982beede01a541f72f603b3d788b89e7b7d3c4fbc1 (good)
;; QUESTION SECTION:
;pc1.ictsc.net.                 IN      A

;; Query time: 3 msec
;; SERVER: 192.168.2.131#53(192.168.2.131)
;; WHEN: Sun Feb 28 15:51:36 JST 2021
;; MSG SIZE  rcvd: 70
```

3. キャッシュサーバで名前解決ができないことがわかったので、コマンド引数に+cd を追加して DNSSEC の検証をせずに名前解決を行います

```shell=
$ dig @192.168.2.131 pc1.ictsc.net +cd

; <<>> DiG 9.11.20-RedHat-9.11.20-5.el8 <<>> @192.168.2.131 pc1.ictsc.net +cd
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 25924
;; flags: qr rd ra cd; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 956
; COOKIE: 67453de86c9abd57bee0655c603b3d852c5c6647f116587d (good)
;; QUESTION SECTION:
;pc1.ictsc.net.                 IN      A

;; ANSWER SECTION:
pc1.ictsc.net.          86329   IN      A       192.168.2.151

;; AUTHORITY SECTION:
ictsc.net.              86338   IN      NS      master-ns.ictsc.net.

;; Query time: 0 msec
;; SERVER: 192.168.2.131#53(192.168.2.131)
;; WHEN: Sun Feb 28 15:51:49 JST 2021
;; MSG SIZE  rcvd: 110
```

名前解決が成功するため，DNSSEC の検証に失敗していることがわかります。

4. クライアントから署名の有効期限を確認します．2 月 10 日に署名の有効期限が切れていることが確認できます

```shell=
$ dig @192.168.2.20 pc1.ictsc.net +dnssec

pc1.ictsc.net 86400 IN RRSIG A 8 3 86400 20210210004000 20210111004000 ...
```

5. 権威サーバで新しい KSK 鍵のみを用いて、ゾーンに再署名を行います．
   (ゾーンの再署名も必要だったのか！)

```shell=
$ sudo su
$ cd /var/named
$ vim ictsc.net.zone

$TTL 1D                                     ;

@         IN  SOA  master-ns.ictsc.net.  root.master-ns.ictsc.net. (  ;
                   2020123101             ;
                   3H                     ;
                   1H                     ;
                   1W                     ;
                   1H )                   ;

          IN  NS   master-ns.ictsc.net.  ;

cache-ns  IN  A    192.168.2.131         ;
master-ns IN  A    192.168.2.20         ;
pc1       IN  A    192.168.2.151         ;
pc2       IN  A    192.168.2.152         ;
pc3       IN  A    192.168.2.153         ;

$INCLUDE "Kictsc.net.+008+63885.key"
$INCLUDE "Kictsc.net.+008+63439.key"
$INCLUDE "Kictsc.net.+008+63829.key" # 古いksk鍵を削除

# -x: KSKの更新のみ
# -o: ゾーンの起点
# -k: 署名鍵
$ dnssec-signzone -x -o ictsc.net -k Kictsc.net.+008+63439.key ictsc.net.zone

Verifying the zone using the following algorithms: RSASHA256.
Zone fully signed:
Algorithm: RSASHA256: KSKs: 1 active, 0 stand-by, 0 revoked
                      ZSKs: 1 active, 0 present, 0 revoked
ictsc.net.zone.signed
```

※ dnssec-signzone: ([参照](https://linux.die.net/man/8/dnssec-signzone)) 6. ゾーンのリロードを行います．

```shell=
$ rndc reload ictsc.net
```

7. クライアント側で KSK 公開鍵を取得し，古いトラストアンカーを削除し新しいトラストアンカーの設定を行います．

```shell=
# 257は現在の有効なKSKを示す
$ dig @192.168.2.20 ictsc.net dnskey | grep 'DNSKEY.*257'
ictsc.net.              86400   IN      DNSKEY  257 3 8 AwEAAePGnJDVqiEhjCRcnYYNP+Pf2DFnJwoj3sTlJwkh2aM1LZR4ajtR sxidDJi59Hf/lcwCBiEnW8eNvpuHz5NfrUTuc/hI/jKI38VkH4m+b68B feNyJtS9IUn8Naln/9r4hQBFCCEHJNmiMo5XnKdD3oEuDSgIsCeP8IOJ c1tlEcimy
fBfijuQleTr7MyoxW3iK0Q7kUuy8kIGelWKMogbUwrFFeBV CNvIAiofQOy7UDkjuGe9UpEXozZ5LNQkrBONzkUvr8Dt3YlhhWWYAjbX W5WzrLiQS9PTr3HMRlOOvTk4XlxQu0LDyqalyuBQnvMMg0AleQ7Q5c+M LU3l96yAg50=

$ sudo vim /var/named/chroot/etc/named.conf
trusted-keys {

        "ictsc.net."  257 3 8 "AwEAAePGnJDVqiEhjCRcnYYNP+Pf2DFnJwoj3sTlJwkh2aM1LZR4ajtR sxidDJi59Hf/lcwCBiEnW8eNvpuHz5NfrUTuc/hI/jKI38VkH4m+b68B feNyJtS9IUn8Naln/9r4hQBFCCEHJNmiMo5XnKdD3oEuDSgIsCeP8IOJ c1tlEcimyfBfijuQleTr7MyoxW3iK0Q7kU
uy8kIGelWKMogbUwrFFeBV CNvIAiofQOy7UDkjuGe9UpEXozZ5LNQkrBONzkUvr8Dt3YlhhWWYAjbX W5WzrLiQS9PTr3HMRlOOvTk4XlxQu0LDyqalyuBQnvMMg0AleQ7Q5c+M LU3l96yAg50=";
};

$ sudo systemctl restart named-chroot
```

8. 最後に、クライアント側でキャッシュサーバを使用して，DNSSEC の検証が成功するかを確認します．

```shell=
$ dig @192.168.2.131 pc1.ictsc.net +dnssec
; <<>> DiG 9.11.20-RedHat-9.11.20-5.el8 <<>> @192.168.2.131 pc1.ictsc.net +dnssec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 50967
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags: do; udp: 956
; COOKIE: 11c4d3fa9bdf7a63dc315b95603de72eaeb7725d8de23282 (good)
;; QUESTION SECTION:
;pc1.ictsc.net.         IN  A
;; ANSWER SECTION:
pc1.ictsc.net.      86258   IN  A   192.168.2.151
pc1.ictsc.net.      86258   IN  RRSIG   A 8 3 86400 20210401061624 20210302061624 63885 ictsc.net. K2/YRHfbne6RIqBzkt0/bZ7T62QTNr52S/SGuD9omC5ClbsJjOydBvXm THSSR0BtxlzbSGVzhggkBDzXcfc7lS8Iv8tbUWDKlvNp+heAo+PhDnWY 8VZWolD2Y2n9HNuBNXhidvIXHyrJbuhtzbamelgnEDx9zKlazeGrbjSZ Kqo=
;; Query time: 0 msec
;; SERVER: 192.168.2.131#53(192.168.2.131)
;; WHEN: Tue Mar 02 16:20:14 JST 2021
;; MSG SIZE  rcvd: 255
```

flags に ad が含まれているため成功。ad は DNSSEC の検証に成功していることを表す。
