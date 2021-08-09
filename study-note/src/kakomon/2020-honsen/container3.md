# いつの間にか復活している君
Edit by 鳥山

[いつの間にか復活している君](https://blog.icttoracon.net/2021/03/16/%e3%81%84%e3%81%a4%e3%81%ae%e9%96%93%e3%81%ab%e3%81%8b%e5%be%a9%e6%b4%bb%e3%81%97%e3%81%a6%e3%81%84%e3%82%8b%e5%90%9b/)

## 使用環境・ツール
- docker
- docker-compose

## 前提条件
- `~/web-server/docker-compose.yml`があり編集可能
    - わざわざ明記しているあたり、これを編集して問題を解決する必要がありそう
    - ⇨ ひっかけポイントだったらしいw そういうパターンもあるのね。
- `docker ps -a`などで確認するとコンテナが起動している

## 問題文でされた操作
- docker-composeを使用してWEBサーバを構築した
- 無事に起動し、WEBページも確認できたのですが、 再起動するとアクセスできなくなってしまった
    - 自動的に再起動するように記述してあるので不可解
- トラブルシュートしているといつの間にかアクセスできるようになります


## バグの内容
踏み台から `$ curl 192.168.17.1` をしても応答がない

## 理想の終了状態
再起動しても踏み台から $ curl -I 192.168.17.1 をするとステータスコード200のレスポンスが返ってくる

## 考えられる検証、修正手順
- `docker build -t test:local . `と Docker でビルドしてみて永遠と転送しているか確認
    - 大量のファイルを裏で Docker daemon が読み込んでいる
    - `.dockerignore` で、大量ファイルのあるディレクトリを除外しておき、docker-compose.json 内で別途マウント
    - 起動の時間によりそう

ref:[docker-compose で一向にビルドがはじまらない、もしくは起動しない。はたまた忘れたころに起動する。](https://qiita.com/KEINOS/items/42aae92d00675c8b0b78)

- Webサーバーにアクセスできない時の確認ポイント(なさそう)
    - コンテナ内のWebサーバーがちゃんと起動して動いているか
        - 以下のようにして確認
        ```
        docker exec -it ＜起動したコンテナ名＞ bash
        curl http://localhost:8080/
        ```
        - エラーが返ってくる、レスポンスが帰ってこない場合はWebサーバーの起動がうまくいっていない
            - 今回の問題は時間が経てば直る問題なのでこれはなさそう
    - localhost以外ではアクセスできるか
        - localhostではなく、実IPアドレスでアクセスしたらどうか？
        - 以下のコマンドでコンテナに割り振られているネットワークアドレスがわかる
        ```
        docker inspect –format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}’ con_name
        curl http://172.17.0.2:8080/
        ```
        - これでうまくいった場合は、ポートなどの問題になる。
    - コンテナで特定のポートが公開されているか
        - ポートの公開を
        ```
        # Dockerfileファイル中に以下のような記述を追加し、8080番ポートを公開する
        EXPOSE 8080
        # composeファイル中にならこんな感じ
        expose:
        - '3306'
        - '8080'
        ```
        - yamlも修正可能なので、なくはないかもしれない。簡単すぎるが。
        - でも、時間が経てば直る問題なのでなさそう
    - 公開されたポートにつながるように設定できているか(ポートフォワード)
        - コンテナで公開されたポートにホストOSから「localhost:8081」のように接続するには、ホストOSの8081番ポートとコンテナの8080番ポートをつないであげる必要があります。
        - localhost:8081 -> コンテナ:8080
            - 「localhost:8081」に来たリクエストを「コンテナ:8080」に転送してあげるようにする
        ```
        docker run -p 8081:8080 –name ＜起動するコンテナ名＞
        ```
        - これも、時間が経てば直る問題なのでなさそう
ref: [Dockerコンテナで起動したサーバにアクセスできないときの確認と対処方法](https://web.plus-idea.net/on/docker-web-server-access-denied/)

### Docker Compose restart の挙動
- ホストOSを起動したタイミングであるアプリケーションを自動で立ち上げたい、 あるいは何らかの問題で落ちた時に、自動で再起動して欲しいというときにrestart使うらしい。
- Docker 及び Compose では、 run/upの restart policy の設定することにより、 コンテナが停止した際の再起動にまつわる設定ができる。

|オプション|意味|
|-|-|
|no	|再起動しない (デフォルト)|
|on-failure[:max-retries]	|プロセスが 0 以外のステータスで終了した場合、 最大:max_retries の分だけ再起動を行う|
|always|	明示的に stop がされない限り、終了ステータスに関係なく常に再起動が行われる|
|unless-stopped	|最後にdocker daemon が起動していた際に ステータスが終了状態だった場合は再起動しない。それ以外はalwaysと同じ。|

ちゃんと再起動の設定はしないといけない。

Q. 従属 コンテナのプロセスはそのまま？ それとも再起動される？<br>
A. そのまま

Q. コンテナ間の接続は再開される？<br>
再起動後も接続は問題なさそう

ref: [Docker Compose restart の挙動](https://junchang1031.hatenablog.com/entry/2016/05/18/000605)

### Dockerデーモン、Dockerコンテナ、及びコンテナ内のサービスアプリの自動起動について
デフォルトは手動なのかな? 再起動の時には自動設定する必要があるっぽい。

#### Dockerデーモンの自動起動
ブート時に自動起動する
```
$ sudo systemctl enable docker
# 他のディストリビューションでは、次のように実行します
$ sudo chkconfig docker on
```

通常時の起動
```
$ sudo systemctl start docker
# 他のディストリビューションでは、次のように実行します
$ sudo service docker start
```

設定確認・状態確認は以下のコマンドで行います。
```
$ systemctl status docker
```

#### Dockerコンテナの自動起動
- restartオプションを利用
    - Dockerが以上終了した場合に自動的に再起動させることが可能
    - さっきの上の表を参考
    - 「always」「unless-stopped」がコンテナの自動起動に利用可能
        - 「always」を指定した場合は、Dockerデーモン終了時のDockerコンテナの状態に関係なく自動起動されます
        - 「unless-stopped」は、Dockerデーモン終了時に停止状態（例えば「docker stop」コマンドにて停止）のコンテナは自動起動されません
- Dockerホストのsystemdを利用する
    - 「docker start」と「docker stop」をsystemdに登録するだけ
    - systemdって起動するときに動いてくれるやつらしい

#### Dockerコンテナ内のサービスの自動起動
- composeファイルに設定がちゃんと書いてあれば特に問題はなさそう

ref: 
- [Dockerデーモンやコンテナ、コンテナ内のサービスの自動起動方法の解説（Linux版）](https://www.memotansu.jp/docker/575/)
- [systemd で Docker の管理・設定](http://docs.docker.jp/v1.11/engine/admin/systemd.html)
- [これからSystemd入門する](https://qiita.com/bluesDD/items/eaf14408d635ffd55a18)

### 修正手順案詳細
- `$ curl 192.168.17.1`と`docker ps -a`を試してみる
- 自動起動されてないようなら、```systemctl status docker```で自動起動を調べる。
- yamlの`restart`オプションを確認して、もしオプションがうまく機能してなさそうなら設定してみる

## 解説

### 原因
デーモンの起動はできているが、dockerの自動起動が出来ていない

### 原因究明方法
初期状態を確認
```
$ curl 192.168.17.1
curl: (7) Failed to connect to 192.168.17.1 port 80: Connection refused
```

コンテナの様子を確認
```
user@docker:~$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                  PORTS                NAMES
4e7db3e7cc97        nginx:latest        "/docker-entrypoint.…"   3 minutes ago       Up Less than a second   0.0.0.0:80->80/tcp   web-server_nginx_1
```
もう一度curlを確認する
```
$ curl 192.168.17.1 -I
HTTP/1.1 200 OK
Server: nginx/1.19.6
Date: Sun, 07 Mar 2021 03:33:11 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 15 Dec 2020 13:59:38 GMT
Connection: keep-alive
ETag: "5fd8c14a-264"
Accept-Ranges: bytes
```
実はdockerコマンドを叩くとデーモンが起動するという罠があります．先ほどのdocker ps -aを見てみるとUp Less than a secondとあり，起動したてなのがわかります

さらに`systemctl status docker`で詳しく確認してみる。
```
ictsc@docker:~$ systemctl status docker
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; disabled; vendor preset: enabled)
     Active: active (running) since Sun 2021-03-07 12:32:53 JST; 9min ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 2176 (dockerd)
      Tasks: 17
     Memory: 114.7M
     CGroup: /system.slice/docker.service
             ├─2176 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
             └─2362 /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 80 -container-ip 172.18.0.2 -container-port 80
              
 
ictsc@docker:~$ systemctl is-enabled docker
disabled
```
↑ でdisabledになっているのが悪かった。

### 解決手順
解放は単純にenableするだけ。
```
systemctl enable docker
```

途中で上手くいっちゃうし、めっちゃ引っかかりそう。restartしてすぐ直るからググってもあんまり出てこなくてこういう問題は意外と厄介かも😅

## その他リンク
時間上説明しなかったものたち。

Webが繋がらないという内容で調べたやつ
- 汎用的に使えそうなやつ: https://web.plus-idea.net/on/docker-web-server-access-denied/
- 全般的に使えそうなやつ: https://qiita.com/amuyikam/items/ef3f8e8e25c557f68f6a

Dockerデーモンが気になって調べたやつ
- [さわって理解する Docker 入門](https://www.ogis-ri.co.jp/otc/hiroba/technical/docker/part6.html)
    - Dockerデーモンは Linux のデーモンプロセスで、Docker Engine API が呼び出されるのを待ち受けています。Dockerデーモンは、呼び出された Docker Engine API に応じて、イメージのビルドやコンテナの起動などを行います。
- [Linux リテラシ - 第4回 デーモン](https://rat.cis.k.hosei.ac.jp/article/rat/linuxliteracy/2005/daemon.html)
    - デーモンはユーザーが意識することがないような裏の部分で動いており、システムを維持したりユーザーにサービスを提供したりといったことを行っています