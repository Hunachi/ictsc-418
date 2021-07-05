# hostnameでつながらない！！

[hostnameでつながらない！！](https://blog.icttoracon.net/2020/11/02/hostname%e3%81%a7%e3%81%a4%e3%81%aa%e3%81%8c%e3%82%89%e3%81%aa%e3%81%84%ef%bc%81%ef%bc%81/)

## 使用環境・ツール
- docker-compose
- wordpress

## 問題文でされた操作
DockerでWordpressの構築を試みる  
それにあたって、docker-compose.ymlファイルで指定したホスト名でデータベースに接続したい

## バグの内容
~/wordpress/docker-compose.ymlを用いて`docker-compose up`をした時にwordpressのコンテナがデータベース接続エラーのログを残して立ち上がらない。

## 理想の終了状態
curl localhost:8000 -Lで正常に200レスポンスが返ってくる。

----

## 考えられる検証、修正手順
### バグの原因を特定する案
- `docker-compose.yml`を確認する
```
version: '3.3'
 
services:
   db: # DO NOT CHANGE THIS LINE
     image: mysql:5.7
     hostname: database # DO NOT CHANGE THIS LINE
     volumes:
       - db_data:/var/lib/mysql
     restart: always
     environment:
       MYSQL_ROOT_PASSWORD: 8MvAMcDAirP8
       MYSQL_DATABASE: wordpress
       MYSQL_USER: wordpress
       MYSQL_PASSWORD: DDzk6ERU33Rc
 
   wp:
     depends_on:
       - db
     image: wordpress:latest
     hostname: wordpress
     ports:
       - "8000:80"
     restart: always
     environment:
       WORDPRESS_DB_HOST: database:3306 # DO NOT CHANGE THIS LINE
       WORDPRESS_DB_USER: wordpress
       WORDPRESS_DB_PASSWORD: DDzk6ERU33Rc
       WORDPRESS_DB_NAME: wordpress
volumes:
    db_data:

```

- [docker composeでwordpressを立ち上げる方法](https://docs.docker.com/samples/wordpress/)のymlファイルとの違いを確認する（特にDB周り）
    - depends_on:で依存関係を指定しているか？
    - mysqlの存在する場所を確認する。位置とか
        ```  
        volumes:
           - db_data:/var/lib/mysql
        ```
- mysqlのパスワードを確認
    - https://qiita.com/go_glzgo/items/3520818659a07bd17839
- データベース自体が破損している
- データベースサーバの異常
    - トラフィック急増による過負荷
- wp-config.php ファイルのユーザー名とパスワードの情報が正しくない
    - WordPressでデータベースに接続する場合、設定情報はwp_config.phpに記述してある。この内容が、docker-compose.ymlで指定している内容と一致しているかを確認する。


- db:3306 のデータベース サーバーに接続できない

### バグの原因が見つかった際の手順


### 修正手順案
- 今回のエラーは、
- `hoge`コマンドを打つ

### 使用した修正手順（壊れた場合ように間違えて踏んでしまった手順は消さずに残しておく）
1. `docker-compose.yml`ファイルを変更した。
```
＊＊＊＊
```

---- 

## 解説

### 原因
- 任意のホスト名でコンテナの名前解決を行うにはエイリアスの設定が必要だった
```
     networks:
       default:
         aliases:
           - database
```

### 解決方法（もっと詳しい説明を書き加える）

docker-compose.ymlを以下のように書き換える。

(`# DO NOT CHANGE THIS LINE`はヒントになるかもしれない)

```
version: '3.3'
 
services:
   db: # DO NOT CHANGE THIS LINE
     image: mysql:5.7
     hostname: database # DO NOT CHANGE THIS LINE
     volumes:
       - db_data:/var/lib/mysql
     restart: always
     environment:
       MYSQL_ROOT_PASSWORD: 8MvAMcDAirP8
       MYSQL_DATABASE: wordpress
       MYSQL_USER: wordpress
       MYSQL_PASSWORD: DDzk6ERU33Rc
     networks:
       default:
         aliases:
           - database　#☆
 
   wp:
     depends_on:
       - db
     image: wordpress:latest
     hostname: wordpress
     ports:
       - "8000:80"
     restart: always
     environment:
       # db:3306でないので動かない　☆
       WORDPRESS_DB_HOST: database:3306 # DO NOT CHANGE THIS LINE
       WORDPRESS_DB_USER: wordpress
       WORDPRESS_DB_PASSWORD: DDzk6ERU33Rc
       WORDPRESS_DB_NAME: wordpress
volumes:
    db_data:
```