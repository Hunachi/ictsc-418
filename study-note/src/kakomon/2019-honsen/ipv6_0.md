# v4v6 移行が終わらない

[v4v6 移行が終わらない](https://blog.icttoracon.net/2020/03/01/v4v6%E7%A7%BB%E8%A1%8C%E3%81%8C%E7%B5%82%E3%82%8F%E3%82%89%E3%81%AA%E3%81%84/)

# 問題

おお久しぶり！！

君が居ないあいだに社内ネットワークを IPv6 only にしておいたんだ。外向きの IP アドレスは v4 しかないけどルーターで NAT64 をしているからインターネットにはつながるようになっているよ。

ただ名前解決ができないのと社内ネットワークにある Web に繋がらなくなっちゃったんだ、これ以上は手が付かないから君がなんとかしてくれないかな？

### 初期状態

-   `ubuntu-1` から `curl https://blog.icttoracon.net/` をしてもつながらない
-   `ubuntu-1` から `curl http://[2403:bd80:c000:900::1]/` をしてもつながらない

### 終了状態

-   `ubuntu-1` から `curl https://blog.icttoracon.net/` をするとステータスコード 200 のレスポンスが返ってくる。
-   `ubuntu-1` から `curl http://[2403:bd80:c000:900::1]/` をするとステータスコード 200 のレスポンスが返ってくる。

### 配点

-   `ubuntu-1` から `curl https://blog.icttoracon.net/` をするとステータスコード 200 のレスポンスが返ってくる
    80%
-   `ubuntu-1` から `curl http://[2403:bd80:c000:900::1]/` をするとステータスコード 200 のレスポンスが返ってくる。
    20%

### 問題文でされた操作

1. 社内ネットワークを IPv6 only にした
2. 外向き IP アドレスは v4 しかないが、ルーターで NAT64 をしている

---

## `https://blog.icttoracon.net/`が返らない原因

#### プリフィックスのある v6 アドレスを DNS で生成されていない

まずは、DNS で IPv6 アドレスを問い合わせてみる

```shell=
dig -t AAAA blog.icttoracon.net +short
```

> NAT64、DNS64 そして通信に介在するネットワーク機器は同じ IPv4-IPv6 変換アドレス用のプリフィクスを使用するよう設定されている必要があります。([参照](https://www.nic.ad.jp/ja/newsletter/No64/0800.html))

今回は外向き IP アドレスは v4 のみなのでプリフィックスのある v6 アドレスが返されないといけない。

## `https://[2403:bd80:c000:900::1]/`が返らない原因

`ping6 2403:bd80:c000:900::1`をしてみる。もし疎通性がなければ、パケットフィルタリングが問題。もし疎通性があれば、前回と同様で web サーバが IPv6 で受け付けてないとか nginx だったら config の不備が問題。

## 考えられる解決方法

#### DNS64 に対応させる

BIND の場合、BIND のバージョンが 9.8 以降か確認し、/etc/bind/named.conf に以下のような記述 DNS64 の設定があるか確認する

```yaml=
options {
    # IPv4は32bitなので96bit付け足してIPv6に対応させる
    dns64 64:ff9b::/96 {
    #     クライアントを指定できるが、anyで良い気がする
        clients { any; };
    };
};
```

`service named restart`をして再起動

UNBOUND の場合は、/etc/unbound/unbound.conf に

```yaml=
server:
    # dns64をつければ後の文字はなんでも良いっぽい
    module-config: "dns64 validator iterator"
    dns64-prefix: 64:ff9b::/96
```

を設定して、`unbound -c unbound.conf`で設定を読み込み

※参照サイト: <br>1.[オライリー DNS64 ページ](https://www.oreilly.com/library/view/dns-and-bind/9781449308025/ch04.html)<br>2.[unbound 公式 DNS64 ドキュメント](https://github.com/NLnetLabs/unbound/blob/master/doc/README.DNS64)

#### Nginx の config を修正

/etc/nginx/nginx.conf の`listen 80;`の下に以下のような記述があるか確認。

```yaml=
# IPv6のポート80でリッスンするよ〜という記述
listen [::]:80;
```

#### パケットフィルタリング

ip6tables という IPv6 の iptables コマンドで確認ができるらしい！([参照](https://linux.die.net/man/8/ip6tables))

```shell=
ip6tables -L
```

アプリケーションに必要なポートを web サーバ側で許可する

```shell=
# TCP80番ポートのアクセス許可
ip6tables -A INPUT -p tcp --dport 80 -j ACCEPT
```

### ※参考※

-   NAT64:IPv6 ホストが IPv4 サーバーと通信することができるようにする技術([わかりやすい仕組み解説](https://www.nic.ad.jp/ja/newsletter/No64/0800.html))

---

## 解説

unbound と Apache2 の設定が適切になされていないことで起こる問題です。

### unbound

`dns64-prefix` は `ubuntu-router` の `JOOL` の設定を参照する必要があります。これを確認すると `dns64-prefix` が `64:ff9b::/96` で有ることがわかります。(`jool global display`コマンドの`pool6`で prefix が確認できそう([参照](https://www.jool.mx/en/usr-flags-global.html)))

また、`blog.icttoracon.net` ではデュアルスタック方式を採用しているおり `dns64-synthall` が no に設定されていると、`blog.icttoracon.net` にもともと設定されている IPv6 アドレスが AAAA レコードとして名前解決されるため、 `ubuntu-1` からアクセスすることができなくなってしまいます。(元から AAAA レコードに何かアドレスが入っていたっぽいな。dns64-synthall のオンオフによる挙動は[ここら辺](https://blog.nic.ad.jp/2016/794/)に載っている)

さらに、この `dns-1` のサーバーは `NAT64` ネットワーク内に設置されているために forward-addr が `8.8.8.8 `になっていると通信が行えません。`forward-addr: 64:ff9b::808:808` に変更する必要があります。(そうだったのか^^;;)

問題が発生している原因は 2 つ存在している。1 つ目は、unbound に DNS64 の設定が正しくされていない点である。これを解決するために `/etc/unbound/unbound.conf.d/dns.conf` を以下のように変更する。([公式の example.conf 参照](https://github.com/fastly/unbound/blob/master/doc/example.conf.in))

```yaml=
server:
# ログレベル
  verbosity: 2
#   pidfileの場所指定
  pidfile: "/var/run/unbound.pid"
#   ログを出すかどうか
  use-syslog: yes
#   さっき書いた通りでdns64を使用する場合はかく
  module-config: "dns64 iterator"
#   prefixを指定する
  dns64-prefix: 64:ff9b::/96
#   yesにするとDNS64やNAT64を通るようになる
  dns64-synthall: yes
  interface: ::0
  access-control: ::0/0 allow

forward-zone:
# `.`にすると全て転送される
  name: "."
#   8.8.8.8のプレフィックス付きIPv6アドレス。
  forward-addr: 64:ff9b::808:808
```

これにより `https://blog.icttoracon.net/` にアクセスできるようになる。

2 つ目の原因は `ubuntu-router` の Web (Apache2) の Listen Address が適切に設定されていないことにある。これを解決するために以下の一行を config に追記する。

```yaml=
Listen [::0]:80
```

### ※参考※

-   NAT64:IPv6 ホストが IPv4 サーバーと通信することができるようにする技術([わかりやすい仕組み解説](https://www.nic.ad.jp/ja/newsletter/No64/0800.html))
-   JOOL:Linux 向け NAT64 オープンソース([参照](https://www.jool.mx/en/))
