# Server Sent Eventsが動作しない‼

[Server Sent Eventsが動作しない‼](https://blog.icttoracon.net/2021/03/16/server-sent-events%e3%81%8c%e5%8b%95%e4%bd%9c%e3%81%97%e3%81%aa%e3%81%84%e2%80%bc/)

## 使用環境・ツール
- nginx

nginxとは（https://wa3.i-3-i.info/word12859.html）：

Apacheのライバル。「Webサーバのソフト」。

普通のコンピュータに「Webサーバのソフト」を入れることによって、Webサーバとしてのお仕事ができるコンピュータになります。

![nginx6](https://wa3.i-3-i.info/img/data/2800/d002859-6.png)

nginxの一番の特徴は、リバースプロキシの機能があることでしょうか。

## 問題文でされた操作

reverse proxyとして、nginxを構築した。

reverse proxyとは（https://wa3.i-3-i.info/word1755.html）：

Webサーバさんの身代わりになってホームページのファイルを返してくれるサーバさんのこと。

![リバースプロキシ19](https://wa3.i-3-i.info/img/data/700/d000755-19.png)

ちなみに、リバースプロキシを使うメリットは

**(1)．身元を隠せる
(2)．[負荷分散](https://wa3.i-3-i.info/word11011.html)ができる**

![リバースプロキシ21](https://wa3.i-3-i.info/img/data/700/d000755-21.png)

## バグの内容

webページ自体は見れるが、sse(server sent events)はうまく動作しない。原因を糾明してsseが動作するようにしてほしい。

server sent eventsとは（[qiita](https://qiita.com/jrsyo/items/abd812ad4b00badf94c8#server-sent-events-%E3%81%A8%E3%81%AF)）：

いわゆる "リアルタイム" イベントを *サーバ -> クライアント* に向けてプッシュできる技術です。

Websocketsとの違い：

Websockets は SSE に比べてより柔軟で強力な機能を持っていますが、実装の複雑さは SSE とは対象的に非常に高コストになりがちです。また大きな違いとして、Websockets では *サーバ <-> クライアント* 間での双方向のリアルタイム通信が可能ですが、SSE はあくまでも *サーバ -> クライアント* (一方向) のデータのプッシュのみをサポートしています。



## 理想の終了状態
Reverse Proxy上で `$ curl -N localhost/event`をしたら、

```
$ curl -N localhost/event 
data: 1 
data: 2
data: 3 
...以下Ctrl+Cが入力されるまで続く
```

----

## 考えられる検証、修正手順

「sse nginx」だけでググったら1番目から6番目までバッファに注意であることがどこかしらに書いてあった。

- https://qiita.com/okumurakengo/items/cbe6b3717b95944083a1#nginx%E3%81%A7%E5%AE%9F%E8%A1%8C
  - > apacheだと特に問題なく動いたのですが、nginxだとうまく行きませんでした、
    > 下のコードだと1秒おきに出力して欲しいのですが、バッファされているようだけのようでタイムアウトしてしまいました。

  - > PHPのファイルに`X-Accel-Buffering: no`を追加するとうまく行きました。

  - > nginxの設定ファイルをいじるという方法でも大丈夫なようです。
    >
    > ```
    > gzip off;
    > proxy_buffering off;
    > fastcgi_keep_conn on;
    > fastcgi_max_temp_file_size 0;
    > fastcgi_buffering off;
    > ```

- https://qiita.com/willow-micro/items/5b245076101460d9dfd6

  - > Nginxに接続すると，SSEの処理だけが絶望的に遅れる
    >
    > 原因：Nginxのリバースプロキシによってバッファリングが行われるため

  - > FlaskからJavascriptの`EventSource`に渡すレスポンスに、ヘッダ`X-Accel-Buffering: no`を付与する

  - > Nginx側の設定に以下を追記すれば良いとの情報がみられる

    > ```
    > proxy_set_header Connection '';
    > proxy_http_version 1.1;
    > chunked_transfer_encoding off;
    > proxy_buffering off;
    > proxy_cache off;
    > ```

----

## 解説

この問題でServer Sent Eventsがうまく動作しなかった原因は、/etc/nginx/nginx.conf内の`proxy_buffering on;`だったことです。nginxのproxy_bufferingが有効な状態だとnginxがバッファリングしてしまい、クライアントがレスポンスを受け取れません。

### 想定解

/etc/nginx/nginx.conf内の`proxy_buffering on;`を`proxy_buffering off;`に置き換えて、nginxを再起動させること

### その他解法

nginxがデフォルトで使用する`http/1.0`はkeep-aliveを使用することは非推奨とされているので、
/etc/nginx/nginx.conf内の`proxy_buffering on;`を`proxy_http_version 1.1;`に置き換えて、nginxを再起動させることでもServer Sent Eventsを動作させることができます。

### 採点基準

部分点は付けず、
Reverse Proxy上で `$ curl -N localhost/event`をしたら、

```
$ curl -N localhost/event` `data: 1` `data: 2` `data: 3` `...以下Ctrl+Cが入力されるまで続く
```

と返ってくるような解答に満点をつけました。