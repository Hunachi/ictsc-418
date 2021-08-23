# 生き返れMariaDB
Edit by 鳥山

[生き返れMariaDB](https://blog.icttoracon.net/2020/03/01/%E7%94%9F%E3%81%8D%E8%BF%94%E3%82%8Cmariadb/)

## 使用環境・ツール
- docker
- MariaDB

## 問題文
- MariaDBのコンテナがすぐに落ちる
- VM上でdocker ps -aをするとmariaDBコンテナが1つ存在している
- VM上でdocker start mariaDB→docker exec -it mariaDB bashをしても落ちて開けない

## 理想の終了状態
- コンテナが継続して起動している状態である
- docker exec -it mariaDB bashでコンテナに入ることができ、入った際にMariaDBのDBictsc_incのusersテーブルを見たとき
```
+----+------------+-----------+------+-------+
| ID | First_Name | Last_Name | Age  | Sex   |
+----+------------+-----------+------+-------+
|  1 | Tarou      | Yamada    |   23 | MAN   |
|  2 | Emi        | Uchiyama  |   23 | WOMAN |
|  3 | Ryo        | Sato      |   25 | MAN   |
|  4 | Yuki       | Tayama    |   22 | WOMAN |
|  5 | Yuto       | Takahashi |   21 | MAN   |
+----+------------+-----------+------+-------+
```
- 社内システムがDBを参照出来るよう、3306番ポートが開いている。

## 配点
- このコンテナが不安定になっている原因を明確な証拠をもとに正しく特定できている(60%)
- コンテナが継続動作するように復旧できている(25%)
- 当該コンテナ上で問題文通りのレコードを参照できる(15%)

----

## 考えられる検証、修正手順

なんとなく、定期的に落ちるならメモリ不足かなと思った

### バグの原因を特定する案
- ログを確認する
  - `docker logs`でログ情報を確認する(確認できるのかな)
  - `docker mysql exited with code 137` こんな感じで出ていたらmysqlが落ちていることがわかる
    - mysqlじゃないけど

- `$ docker stats` してみる

こんな流れでメモリ不足が出る
1. dockerでメモリを過剰に使用
2. mysqlのdockerがメモリ不足で落ちる
3. ホストのoom-killerも動き、何故かmysqlが狙われてkillされる

### メモリの割り当て
```
docker-machine inspect
```
以下のコマンドでメモリの割り当てをする
```
docker run -m 1024m hoge /bin/bash
```
https://qiita.com/niisan-tokyo/items/2d7d21aeb4e25f7a7bbe

### oom-killerのkillを阻止する
こんな感じでkillの確認ができる
```
$ sudo cat /var/log/messages | grep Killed
Oct  1 11:11:54 ip-xx-xx-xx-xx kernel: [1983378.957901] Killed process 5789 (ruby) total-vm:4957320kB, anon-rss:2717004kB, file-rss:0kB
```

危なそうなプロセスの確認
```
$ dstat --top-oom
--out-of-memory---
    kill score
mysqld        484
mysqld        484
mysqld        484
```

修正は`/proc/PID/oom_adj`に（優先度低）-16から+15（優先度高）の値を設定することができる

-----
## 解説

## 原因の特定方法
1. docker inspectコマンドを使用する  
docker inspectコマンドは、dockerコンテナの設定情報を参照することが出来るコマンドです。これを見ると、コンテナがどのような設定で動作しているのかが分かります。しかし、この設定データの量は膨大で、全てを見るには時間がかかってしまいます。 例えばですが、この問題の原因のメモリについて調べたい際、など、grepコマンドなどを使い検索すると、
```
$ docker inspect mariaDB | grep Memory
"Memory": 4194304,
           "KernelMemory": 0,
           "MemoryReservation": 0,
           "MemorySwap": -1,
           "MemorySwappiness": null,
```
といった値が出てきます。ここに出た"Memory": 4194304という値はバイト表記でして、MBに換算すると約4MBということが分かります。設定されていた値と同じですね。

2. docker statsコマンドを使用する  
docker statsコマンドは、コンテナのリソース使用状況を表示するコマンドです。Linuxのtopコマンドに似たような機能を持っています。 ここで、コンテナのリソース使用状況を知ることが出来ます。

```
$ docker stats
CONTAINER ID        NAME                CPU %               MEM USAGE / LIMIT   MEM %               NET I/O             BLOCK I/O           PIDS
eb99786557f7        mariaDB             8.44%               3.664MiB / 4MiB     91.60%              656B / 0B           1.89GB / 0B         3
```
こちらでもMEM USAGE / LIMITの欄で3.664MiB / 4MiBと見えており、メモリが4MB制限であること、そして使用中のメモリが逼迫している状態であることから不安定になる原因であることが推定出来ます。


## 解決手順
2パターン。
1. docker updateコマンドを使用しリソース設定を更新する 
docker updateコマンドはコンテナの設定を更新するためのコマンドです。 コンテナの設定をコンテナを作り直すことなく変更することが可能です。 例として、問題のコンテナのメモリ制限を4MBから1GBへと緩和します。例えば、 `docker update --memory 1G mariaDB`
このコマンドを実行する事により、メモリ上限を4MBから1GBに変更することが出来ます。 

```
$ docker inspect mariaDB | grep Memory
"Memory": 1073741824,
"KernelMemory": 0,
"MemoryReservation": 0,
"MemorySwap": -1,
"MemorySwappiness": null,
```

となり、Memory": 1073741824(byte)は約1GBですので、1GBへと緩和されたことが分かります


2. 同じ設定のコンテナを作り直す  

解法の一つとして、同じ設定のMariaDBコンテナを再作成する方法があります。 `docker run -v mariaVOL:/var/lib/mysql -d --name mariaDB -e MYSQL_ROOT_PASSWORD=MariaPass -p 3306:3306 -d mariadb:latest` などで、メモリ制限を無くしたコンテナを作成します。 ただし、MariaDBのデータベースはボリュームmariaVOLにマウントされているという点に注意しなければなりません。mariaVOLへコンテナをマウントしないと、コンテナに入ってもデータベースを参照することが出来なくなってしまいます。

上記の2つの設定のどちらかを適用し、

```
$ docker exec -it mariaDB bash
$ mysql -u root -p ${MYSQL_ROOT_PASSWORD}

> use ictsc_inc;
> select * from users;
// selectの結果が表示される
```

