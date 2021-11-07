# Welcome to Nginx のページを表示したい！
解いた人:[momom-i](https://github.com/momom-i)

参照した問題・解説のサイト:[Welcome to Nginx のページを表示したい！](https://blog.icttoracon.net/2019/12/10/ictsc2019-%e4%ba%8c%e6%ac%a1%e4%ba%88%e9%81%b8-%e5%95%8f%e9%a1%8c%e8%a7%a3%e8%aa%ac-welcome-to-nginx%e3%81%ae%e3%83%9a%e3%83%bc%e3%82%b8%e3%82%92%e8%a1%a8%e7%a4%ba%e3%81%97%e3%81%9f%e3%81%84%ef%bc%81/)

# 問題

あなたはローカルネットワーク上に web サーバを構築し、IPv6 アドレスを使用して web ページに接続できるようにセットアップをしています。
web ページへは`http://nginx.icttoracon.net`でアクセスできるようにしたいです。

nginx のホストには、すでに nginx のパッケージをインストール済みです。
CSR1000V と nginx のホストには固定で IPv6 アドレスを割り当てました。
クライアント(VNC Server)に IPv6 アドレスが自動設定されるように、CSR1000V には SLAAC の設定を行いました。

しかし、クライアント(VNC Server)のブラウザから`http://nginx.icttoracon.net`にアクセスしても Welcome to Nginx のページを表示させることができません。
このトラブルを解決し、Welcome to Nginx のページを表示させてください。

クライアントが増えても自動でアクセスできるよう、設定変更は CSR1000V と nginx ホストのみとしてください。
DNS サーバは CSR1000V を使用します。
各ノードには ssh/telnet 用に IPv4 アドレスが設定されていますので必要に応じて使用してください。
予選終了後に実環境で採点されるので、スコアサーバでの解答は不要です。

### 接続環境

| Host     | Protocol | IPv4 address |  User/Pass  |
| :------- | :------: | :----------: | :---------: |
| CSR1000V |  telnet  | 192.168.0.1  | admin/admin |
| nginx    |   ssh    | 192.168.1.2  | admin/admin |

### 問題文でされた操作

1. CSR1000V と nginx のホストに固定で IPv6 アドレスを割り当て
2. CSR1000V には SLAAC の設定

### バグの内容

`http://nginx.icttoracon.net`にアクセスしても Welcome to Nginx のページを表示させることができない

### 理想の終了状態

Welcome to Nginx のページを表示

## 補足事項

-   各ノードには ssh/telnet 用に IPv4 アドレスが設定されていますので必要に応じて使用
-   設定変更は CSR1000V と nginx ホストのみ

---

## 考えられる原因

#### まず下記コマンドでクライアントから`nginx.icttoracon.net`が引けるか、IPv6 アドレスが引けるか調べる

```shell=
# AAAAレコードはホスト名に対するIPv6アドレスが登録されている(ちなみにAAAAレコードはクアッドエーレコードって読むらしい！)
# +shortオプションで簡単にIPv6アドレスが登録されてるかどうか以外の情報を省ける
dig -t AAAA nginx.icttoracon.net +short
```

-   もし IPv4 でも引けないなら DNS の設定問題。(問題文的に IPv4 では引けそう？)IPv4 では引けるが IPv6 アドレスが引けなかったら、DNS のゾーンファイルに IPv6 の設定がなされてない。もしどちらも登録されていたら、nginx の問題で config の書き方不備。

#### クライアントから ping で IPv6 のアドレスにつながるかやってみる

```shell=
# ping6は `ping -6`と同じ
ping6 nginx.icttoracon.net
```

-   もし通らなければ、SLAAC の自動 ip 割り当てあたりやパケットフィルタリングが問題。通るなら上記の問題になってくる気がする。

## 考えられる解決方法

#### DNS のゾーンファイルの不備を修正

```yaml=
# IPv6は省略法で書けるらしい
nginx.icttoracon.net. IN AAAA fc01::2
```

#### Nginx の config を修正

/etc/nginx/nginx.conf の`listen 80;`の下に以下のような記述があるか確認

```yaml=
# IPv6のポート80でリッスンするよ〜という記述
listen [::]:80;
```

#### SLAAC の自動 IP 割り当て確認

基本的に[このページ](https://www.n-study.com/ipv6-detail/cisco-ipv6-address-configuration/)を参照した

```shell=
show ipv6 address
```

で確認できる。もし割り当てがおかしい場合は、Cisco のグローバルコンフィグレーションモード(Cisco のルーターの特権モード。全体の設定に関わるようなことを行う時に入るモード)に`configure terminal`で入って、

```shell=
(config)#ipv6 unicast-routing
# インターフェイス名を指定する。スロットは固定で0みたい
(config)#interface gigabitethernet 0/1
(config-if)#ipv6 enable
```

※Cisco のインターフェイスについては[こちら](https://www.n-study.com/cisco-basic/cisco-interface/)

#### パケットフィルタリング

ip6tables という IPv6 の iptables コマンドで確認ができるらしい！([参照](https://linux.die.net/man/8/ip6tables))

```shell=
ip6tables -L
```

もし 80 番ポートの設定がなければ、今回はクライアント側は変更しないから nginx で以下のコマンドで 80 番ポートの接続を許可する

```shell=
# TCP80番ポートのアクセス許可
ip6tables -A INPUT -p tcp --dport 80 -j ACCEPT
```

### ※参考※

-   CSR1000V(Cisco Cloud Services Router 1000V):ソフトウェアルータ([参照](https://www.cisco.com/c/en/us/products/collateral/routers/cloud-services-router-1000v-series/data_sheet-c78-733443.html))
-   SLAAC(スラーク): <br>IPv6 アドレスを自動設定する技術の一つで、アドレスを発行するサーバなどを用意しなくても当該ネットワーク内のアドレスをホスト自身が設定する方式([概要説明参照 URL](https://e-words.jp/w/SLAAC.html), [具体的な挙動のわかりやすい参照 URL](https://www.n-study.com/ipv6-detail/ipv6-address-configuration/)(このページの「SLAAC による IPv6 アドレスの設定」))
-   VNC: ネットワークを通じて別のコンピュータに接続し、そのデスクトップ画面を呼び出して操作することができるリモートデスクトップソフトの一つ([参照](https://e-words.jp/w/VNC.html))

-   IPv6 の記述法について: <br>例. fc01::2 = fc01:0000:0000:0000:0000:0000:0000:0002/64([参照](https://www.infraexpert.com/study/ipv6z2.html#:~:text=IPv4%E3%82%A2%E3%83%89%E3%83%AC%E3%82%B9%E3%81%AF%E3%80%8132%E3%83%93%E3%83%83%E3%83%88,%E5%88%86%E3%81%9116%E9%80%B2%E6%95%B0%E3%81%A7%E8%A1%A8%E8%A8%98%E3%80%82))

---

## 解説

本問題には 3 つの原因があります。
順を追って調べてみましょう。

#### 疎通性確認

まずはクライアントマシンから nginx ホストまで IPv6 で疎通性があるか確認してみます。
簡単な確認ではありますが、トラブル原因のレイヤをある程度限定できます。

```shell=
# ping6 ドメイン名ってやってたけど、普通にIPv6アドレスでpingするべきだったな^^;
ubuntu@ICTSC-VNC:~$ ping fc01::2 -c 4
PING fc01::2(fc01::2) 56 data bytes
64 bytes from fc01::2: icmp_seq=1 ttl=63 time=0.887 ms
64 bytes from fc01::2: icmp_seq=2 ttl=63 time=0.607 ms
64 bytes from fc01::2: icmp_seq=3 ttl=63 time=0.802 ms
64 bytes from fc01::2: icmp_seq=4 ttl=63 time=0.699 ms

--- fc01::2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3040ms
rtt min/avg/max/mdev = 0.607/0.748/0.887/0.110 ms
```

コマンドの結果から本問題は初期状態で IPv6 の疎通性があることが確認できます。

#### 名前解決

名前解決ができるかどうか試してみましょう。
ドメイン名から IPv6 アドレスを取得するには AAAA レコードを参照します。
例として AAAA レコードを取得するコマンドを以下に示します。

```shell=
ubuntu@ICTSC-VNC:~$ dig nginx.icttoracon.net AAAA

; <<>> DiG 9.11.3-1ubuntu1.11-Ubuntu <<>> nginx.icttoracon.net AAAA
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 40684
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;nginx.icttoracon.net.        IN  AAAA

;; Query time: 89 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
;; WHEN: Wed Dec 04 22:46:46 JST 2019
;; MSG SIZE  rcvd: 49
ubuntu@ICTSC-VNC:~$
```

AAAA レコードは取得できていません。
問題文で DNS サーバは CSR とされていますが、クライアントはどこを参照しているのでしょうか。

```shell=
# grep -v: invert match
# grep -v "^#": #で始まるもの以外を取り出す = コメントアウト部分以外を取り出す
ubuntu@ICTSC-VNC:~$ cat /etc/resolv.conf | grep -v "^#"

nameserver 127.0.0.53
options edns0
search localdomain
# この/run/systemd/resolve/resolv.confは/etc/resolv.confのシンボリックリンク先
# `nameserver fc00::1`があるはず
ubuntu@ICTSC-VNC:~$ cat /run/systemd/resolve/resolv.conf | grep -v "^#"

nameserver 133.242.0.3
nameserver 133.242.0.4
search localdomain
ubuntu@ICTSC-VNC:~$
```

IPv4 で DNS サーバを受け取っているようですが、CSR の IP アドレスではありません。
１つ目の原因は DNS サーバ（CSR）が参照できていないことです。

問題文からクライアントの設定変更ではなくルータの設定変更で対応する方針であることがわかります。
クライアントの設定と CSR の設定を確認すると、クライアントの IPv6 アドレスは RA を用いた自動設定であることがわかります。
ただし DNS サーバのアドレスが配布されていません。
RA で IPv6 アドレスが設定されている場合は、以下の 2 つの方法で DNS サーバを配布することができます。

-   ステートレス DHCPv6
-   RA を用いた DNS 配布(RFC8106)

例として RA のみで DNS の配布を行います。
IOS-XE(Cisco の OS([参考](https://www.cisco.com/c/ja_jp/products/ios-nx-os-software/ios-xe/index.html)))のコマンドリファレンスを参照すると、以下の設定で DNS の配布が行えそうです。

```shell=
# conf t: さっきのconfigure terminalの省略形
csr1000v#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
# int gi 1 = interface gigabitethernet 0/1と同じ
csr1000v(config)#int gi 1
csr1000v(config-if)#ipv6 nd ra dns server fc00::1
```

([ipv6 nd ra dns server fc00::1 の説明](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipv6/command/ipv6-cr-book/ipv6-i3.html#wp3145726977))
クライアントで確認してみます。

```shell=
ubuntu@ICTSC-VNC:~$ cat /run/systemd/resolve/resolv.conf | grep -v "^#"

nameserver 133.242.0.3
nameserver 133.242.0.4
nameserver fc00::1
search localdomain
ubuntu@ICTSC-VNC:~$ dig nginx.icttoracon.net AAAA

; <<>> DiG 9.11.3-1ubuntu1.11-Ubuntu <<>> nginx.icttoracon.net AAAA
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 35863
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;nginx.icttoracon.net.        IN  AAAA

;; ANSWER SECTION:
nginx.icttoracon.net.    10  IN  AAAA    fc01::2

;; Query time: 12 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
;; WHEN: Wed Dec 04 22:51:49 JST 2019
;; MSG SIZE  rcvd: 77

ubuntu@ICTSC-VNC:~$
```

DNS サーバとして fc00::1 が設定され、AAAA レコードが正しく参照できています。

#### nginx 設定

IPv6 の疎通性があり、名前解決も行えているので一旦クライアントの作業を終え、nginx ホストを確認してみます。
まずは 80 ポートの使用状況を確認してみます。

```shell=
[admin@nginx ~]$ sudo lsof -i:80
COMMAND  PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nginx   1421  root    6u  IPv4   9815      0t0  TCP *:http (LISTEN)
nginx   1422 nginx    6u  IPv4   9815      0t0  TCP *:http (LISTEN)
```

type を見ると IPv4 となっており、nginx が IPv6 アドレスで待ち受けていないことがわかります。
2 つ目の原因は nginx は IPv6 アドレスで待ち受けていないことです。
nginx が IPv6 アドレスで待ち受けるよう、設定を変更します。

```shell=
server {
    listen       80;
+   listen       [::]:80;
    server_name  localhost;

--- snip ---
# reload忘れてた^^;
[admin@nginx ~]$ sudo nginx -s reload
[admin@nginx ~]$ sudo nginx -s reload
[admin@nginx ~]$ sudo lsof -i:80
COMMAND  PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nginx   1421  root    6u  IPv4   9815      0t0  TCP *:http (LISTEN)
nginx   1421  root   10u  IPv6  10932      0t0  TCP *:http (LISTEN)
nginx   1540 nginx    6u  IPv4   9815      0t0  TCP *:http (LISTEN)
nginx   1540 nginx   10u  IPv6  10932      0t0  TCP *:http (LISTEN)
```

#### フィルタリング設定

nginx が IPv6 で待ち受ける状態となりました。
しかしまだクライアントからアクセスができません。
nginx ホストのディストリビューションを確認してみます。

```shell=
[admin@nginx ~]$ ls /etc | grep release
centos-release
redhat-release
system-release
system-release-cpe
[admin@nginx ~]$ cat /etc/centos-release
CentOS release 6.10 (Final)
[admin@nginx ~]$
```

CentOS6.10 であるため、フィルタリングは iptables で行っていると予想されます。(ここまで考えてなかった^^;)
iptables のルールを確認してみましょう。

```shell=
[admin@nginx ~]$ sudo iptables -L INPUT
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     all  --  anywhere             anywhere            state RELATED,ESTABLISHED
ACCEPT     icmp --  anywhere             anywhere
ACCEPT     all  --  anywhere             anywhere
ACCEPT     tcp  --  anywhere             anywhere            state NEW tcp dpt:ssh
ACCEPT     tcp  --  anywhere             anywhere            state NEW tcp dpt:http
REJECT     all  --  anywhere             anywhere            reject-with icmp-host
```

一見すると問題が無いように見えますが、クライアントは Welcome to Nginx のページにアクセスできません。
それもそのはず、iptables は IPv4 のフィルタリング設定だからです。
実は初期状態から IPv4 で 80 ポートは許可されており、クライアントは IPv4 を用いて Welcome to Nginx のページを表示させることはできてました。

IPv6 のフィルタリングは ip6table で行います。
ip6table のルールを確認すると、80 ポートのアクセスを許可しているルールが無いことがわかります。

```shell=
[admin@nginx ~]$ sudo ip6tables -L INPUT
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     all      anywhere             anywhere            state RELATED,ESTABLISHED
ACCEPT     ipv6-icmp    anywhere             anywhere
ACCEPT     all      anywhere             anywhere
ACCEPT     udp      anywhere             fe80::/64           state NEW udp dpt:dhcpv6-client
ACCEPT     tcp      anywhere             anywhere            state NEW tcp dpt:ssh
REJECT     all      anywhere             anywhere            reject-with icmp6-adm-prohibited

```

3 つ目の原因は IPv6 の 80 ポートが拒否されていることです。
問題文には恒久的な設定ではなくて構わないとしか記載されていないので、80 ポートを許可する方法か、プロセスを停止する方法があります。
問題としてはどちらで行ってもいいですが、望ましいのは 80 ポートを許可する方法です。
ip6tables で 80 ポートを許可します。

```shell=
# -m state: パケットの状態
# -m state NEW: 新規接続を対象にするという意味
# -I INPUT 6: 6番目に挿入
[admin@nginx ~]$ sudo ip6tables -I INPUT 6 -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT
[admin@nginx ~]$ sudo ip6tables -I INPUT 6 -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT
[admin@nginx ~]$ sudo ip6tables -L INPUT
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     all      anywhere             anywhere            state RELATED,ESTABLISHED
ACCEPT     ipv6-icmp    anywhere             anywhere
ACCEPT     all      anywhere             anywhere
ACCEPT     udp      anywhere             fe80::/64           state NEW udp dpt:dhcpv6-client
ACCEPT     tcp      anywhere             anywhere            state NEW tcp dpt:ssh
ACCEPT     tcp      anywhere             anywhere            state NEW tcp dpt:http
REJECT     all      anywhere             anywhere            reject-with icmp6-adm-prohibited

```

クライアントでアクセスしてみると、Welcome to Nginx のページが表示されます。
