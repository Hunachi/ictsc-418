# MySQLの復旧をお願いします！！
解いた人:[Hunachi](https://twitter.com/_hunachi)

参照した問題・解説のサイト:[MySQLの復旧をお願いします！！](https://blog.icttoracon.net/2019/12/10/ictsc2019-%e4%ba%8c%e6%ac%a1%e4%ba%88%e9%81%b8-%e5%95%8f%e9%a1%8c%e8%a7%a3%e8%aa%ac-mysql%e3%81%ae%e5%be%a9%e6%97%a7%e3%82%92%e3%81%8a%e9%a1%98%e3%81%84%e3%81%97%e3%81%be%e3%81%99%ef%bc%81%ef%bc%81/)

## 使用環境・ツール

### 環境
- IPアドレス: 192.168.0.1
- ユーザー: admin
- パスワード: USerPw@19
- DBユーザー: root
- DBパスワード: root

### 状況
- このMySQLは毎日定時に`sysbench database`のバックアップを取得していて(コンテスト問題の作成上truncate table文が実行された日まで)、偶然truncate文が実行される(数分)前にこの日のバックアップが完了していた
- バックアップは以下のコマンドで取得されている
- `mysqldump --opt --single-transaction --master-data=2 --default-character-set=utf8mb4 --databases sysbench > /root/backup/backup.dump`
- `mysql -u root -p < /root/backup/backup.dump`でバックアップが取得された時点に復旧できる
- adminユーザからsudo suすることでrootユーザから操作してください

## 問題

### 問１

`truncate table sbtest3;`というクエリが実行された日時を`yymmdd HH:MM:SS`のフォーマットで報告してください。
また、どのようにこの日時を特定したかを説明してください。

### 問２
`truncate table`が実行される直前の状態(`truncate table`が実行される1つ前のクエリが実行された状態)にデータを復旧し、復旧後`checksum table sysbench.sbtest3;`の結果を報告してください。
また、データの復旧に必要な手順を説明してください。

----

## 技術調査

### 問１について

- 実行されたmysqlコマンドの履歴を表示する方法
`cat ~/.mysql_history`
で実行されたmysqlコマンドの履歴を表示することができるはず。[参考](https://codehero.jp/mysql/7818031/sql-command-to-display-history-of-queries)

これで試した結果(dockerで環境作ってクエリを適当に打った)
```
# cat ~/.mysql_history
_HiStOrY_V2_
show\040databases;
quit;
```

\040はASCIIコードのスペースだけど、見づらいのでこれはスペースで表示させるようにする。

```
# sed "s/\\\040/ /g"  ~/.mysql_history
_HiStOrY_V2_
show databases;
quit;
```

[sedコマンドについて参考になるサイト](https://hydrocul.github.io/wiki/commands/sed.html)

日時が表示されてないのでだめ！

どうやらデフォで日時を取得する方法ないかも。。? 　
https://forums.mysql.com/read.php?10,400933
https://forums.mysql.com/read.php?10,400933,401057#msg-401057

そこで、
`general_log`が有効になっているかを確認する。
`show variables like 'general_log';`
もし、

```mysql> show variables like 'general_log';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| general_log   | ON    |
+---------------+-------+
1 row in set (0.00 sec)
```
ならラッキー、いける。
OFFなら自分にはお手上げ😭🙌

ONだったら、
`show variables like 'general_log_file';`
でわかるファイルをみてみる。

**実験した**
1. デフォではOFFだったので、
`mysql> set global general_log = 'ON';`を設定。
2. 色々コマンドを打つ。
3. `cat /var/lib/mysql/${general_log_file's value}.log`
4. 表示される例。
```
/usr/sbin/mysqld, Version: 8.0.25 (MySQL Community Server - GPL). started with:
Tcp port: 3306  Unix socket: /var/run/mysqld/mysqld.sock
Time                 Id Command    Argument
2021-08-16T08:48:21.048791Z	   15 Query	SELECT DATABASE()
2021-08-16T08:48:21.049349Z	   15 Init DB	hunadb
2021-08-16T08:48:21.052633Z	   15 Query	show databases
2021-08-16T08:48:21.053892Z	   15 Query	show tables
2021-08-16T08:48:21.055342Z	   15 Field List	piyo 
2021-08-16T08:48:32.444434Z	   15 Quit	
2021-08-16T08:48:34.020718Z	   16 Connect	root@localhost on  using Socket
2021-08-16T08:48:34.021351Z	   16 Query	select @@version_comment limit 1
2021-08-16T08:48:36.401359Z	   16 Query	show variables like 'general_log'
2021-08-16T08:48:46.839490Z	   16 Query	create table hunadb.piyo (id int, cost int)
2021-08-16T08:49:06.218438Z	   16 Query	create table hunadb.hiyo (id int, cost int)
2021-08-16T08:49:43.064247Z	   16 Query	truncate table hunadb.piyo
2021-08-16T08:49:48.068001Z	   16 Quit
```

↑の日時を解答として提出すれば良さそう！！

### 問2について

状況にて、バックアップの復旧方法を教えてくれてるのでそれをする。(括弧内は自分が打ったコマンド。ログも実際も私の物。)

1. `mysqldump --opt --single-transaction --master-data=2 --default-character-set=utf8mb4 --databases sysbench > /root/backup/backup.dump` (を打つべきだけどこの時は手を抜いて`mysqldump hunadb > dump.sql`)
2. `mysql -u root -p < /root/backup/backup.dump`(実際に打ったのは、`mysql hunadb < dump.sql`)
3. `.mysql_history`を確認してみる。
```
/usr/sbin/mysqld, Version: 8.0.25 (MySQL Community Server - GPL). started with:
Tcp port: 3306  Unix socket: /var/run/mysqld/mysqld.sock
Time                 Id Command    Argument
2021-08-16T08:48:21.048791Z	   15 Query	SELECT DATABASE()
2021-08-16T08:48:21.049349Z	   15 Init DB	hunadb
2021-08-16T08:48:21.052633Z	   15 Query	show databases
2021-08-16T08:48:21.053892Z	   15 Query	show tables
2021-08-16T08:48:21.055342Z	   15 Field List	piyo 
2021-08-16T08:48:32.444434Z	   15 Quit	
2021-08-16T08:48:34.020718Z	   16 Connect	root@localhost on  using Socket
2021-08-16T08:48:34.021351Z	   16 Query	select @@version_comment limit 1
2021-08-16T08:48:36.401359Z	   16 Query	show variables like 'general_log'
2021-08-16T08:48:46.839490Z	   16 Query	create table hunadb.piyo (id int, cost int)
2021-08-16T08:49:06.218438Z	   16 Query	create table hunadb.hiyo (id int, cost int)
2021-08-16T08:49:43.064247Z	   16 Query	truncate table hunadb.piyo
2021-08-16T08:49:48.068001Z	   16 Quit	
2021-08-16T09:43:42.989880Z	   17 Connect	root@localhost on  using Socket
2021-08-16T09:43:42.990214Z	   17 Query	select @@version_comment limit 1
2021-08-16T10:08:24.906898Z	   17 Query	SELECT * FROM msdb.dbo
2021-08-16T10:08:34.540831Z	   17 Query	show databases
2021-08-16T10:10:22.024284Z	   17 Quit	

--- ここでバックアップを取った --- （ mysqldump hunadb > dump.sql ) 

2021-08-16T10:10:31.893768Z	   18 Connect	root@localhost on  using Socket
2021-08-16T10:10:31.893799Z	   18 Connect	Access denied for user 'root'@'localhost' (using password: NO)
2021-08-16T10:11:05.935347Z	   19 Connect	root@localhost on  using Socket
2021-08-16T10:11:05.935406Z	   19 Connect	Access denied for user 'root'@'localhost' (using password: NO)
2021-08-16T10:11:29.099579Z	   20 Connect	root@localhost on  using Socket
2021-08-16T10:11:29.099723Z	   20 Query	/*!40100 SET @@SQL_MODE='' */
2021-08-16T10:11:29.099984Z	   20 Query	/*!40103 SET TIME_ZONE='+00:00' */
2021-08-16T10:11:29.100167Z	   20 Query	/*!80000 SET SESSION information_schema_stats_expiry=0 */
2021-08-16T10:11:29.100304Z	   20 Query	SET SESSION NET_READ_TIMEOUT= 86400, SESSION NET_WRITE_TIMEOUT= 86400
2021-08-16T10:11:29.101544Z	   20 Query	SHOW VARIABLES LIKE 'gtid\_mode'
2021-08-16T10:11:29.103106Z	   20 Query	SELECT LOGFILE_GROUP_NAME, FILE_NAME, TOTAL_EXTENTS, INITIAL_SIZE, ENGINE, EXTRA FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'UNDO LOG' AND FILE_NAME IS NOT NULL AND LOGFILE_GROUP_NAME IS NOT NULL AND LOGFILE_GROUP_NAME IN (SELECT DISTINCT LOGFILE_GROUP_NAME FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'DATAFILE' AND TABLESPACE_NAME IN (SELECT DISTINCT TABLESPACE_NAME FROM INFORMATION_SCHEMA.PARTITIONS WHERE TABLE_SCHEMA IN ('hunadb'))) GROUP BY LOGFILE_GROUP_NAME, FILE_NAME, ENGINE, TOTAL_EXTENTS, INITIAL_SIZE ORDER BY LOGFILE_GROUP_NAME
2021-08-16T10:11:29.110332Z	   20 Query	SELECT DISTINCT TABLESPACE_NAME, FILE_NAME, LOGFILE_GROUP_NAME, EXTENT_SIZE, INITIAL_SIZE, ENGINE FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'DATAFILE' AND TABLESPACE_NAME IN (SELECT DISTINCT TABLESPACE_NAME FROM INFORMATION_SCHEMA.PARTITIONS WHERE TABLE_SCHEMA IN ('hunadb')) ORDER BY TABLESPACE_NAME, LOGFILE_GROUP_NAME
2021-08-16T10:11:29.112213Z	   20 Query	SHOW VARIABLES LIKE 'ndbinfo\_version'
2021-08-16T10:11:29.114053Z	   20 Init DB	hunadb
2021-08-16T10:11:29.114225Z	   20 Query	show tables
2021-08-16T10:11:29.115254Z	   20 Query	LOCK TABLES `hiyo` READ /*!32311 LOCAL */,`piyo` READ /*!32311 LOCAL */
2021-08-16T10:11:29.116322Z	   20 Query	show table status like 'hiyo'
2021-08-16T10:11:29.117349Z	   20 Query	SET SQL_QUOTE_SHOW_CREATE=1
2021-08-16T10:11:29.117507Z	   20 Query	SET SESSION character_set_results = 'binary'
2021-08-16T10:11:29.117621Z	   20 Query	show create table `hiyo`
2021-08-16T10:11:29.117877Z	   20 Query	SET SESSION character_set_results = 'utf8mb4'
2021-08-16T10:11:29.118037Z	   20 Query	show fields from `hiyo`
2021-08-16T10:11:29.119264Z	   20 Query	show fields from `hiyo`
2021-08-16T10:11:29.120115Z	   20 Query	SELECT /*!40001 SQL_NO_CACHE */ * FROM `hiyo`
2021-08-16T10:11:29.120363Z	   20 Query	SET SESSION character_set_results = 'binary'
2021-08-16T10:11:29.120561Z	   20 Query	use `hunadb`
2021-08-16T10:11:29.120795Z	   20 Query	select @@collation_database
2021-08-16T10:11:29.121025Z	   20 Query	SHOW TRIGGERS LIKE 'hiyo'
2021-08-16T10:11:29.122437Z	   20 Query	SET SESSION character_set_results = 'utf8mb4'
2021-08-16T10:11:29.122713Z	   20 Query	SET SESSION character_set_results = 'binary'
2021-08-16T10:11:29.122975Z	   20 Query	SELECT COLUMN_NAME,                       JSON_EXTRACT(HISTOGRAM, '$."number-of-buckets-specified"')                FROM information_schema.COLUMN_STATISTICS                WHERE SCHEMA_NAME = 'hunadb' AND TABLE_NAME = 'hiyo'
2021-08-16T10:11:29.123610Z	   20 Query	SET SESSION character_set_results = 'utf8mb4'
2021-08-16T10:11:29.123849Z	   20 Query	show table status like 'piyo'
2021-08-16T10:11:29.124940Z	   20 Query	SET SQL_QUOTE_SHOW_CREATE=1
2021-08-16T10:11:29.125141Z	   20 Query	SET SESSION character_set_results = 'binary'
2021-08-16T10:11:29.125352Z	   20 Query	show create table `piyo`
2021-08-16T10:11:29.125741Z	   20 Query	SET SESSION character_set_results = 'utf8mb4'
2021-08-16T10:11:29.125958Z	   20 Query	show fields from `piyo`
2021-08-16T10:11:29.127051Z	   20 Query	show fields from `piyo`
2021-08-16T10:11:29.128547Z	   20 Query	SELECT /*!40001 SQL_NO_CACHE */ * FROM `piyo`
2021-08-16T10:11:29.128818Z	   20 Query	SET SESSION character_set_results = 'binary'
2021-08-16T10:11:29.128976Z	   20 Query	use `hunadb`
2021-08-16T10:11:29.129195Z	   20 Query	select @@collation_database
2021-08-16T10:11:29.129379Z	   20 Query	SHOW TRIGGERS LIKE 'piyo'
2021-08-16T10:11:29.130227Z	   20 Query	SET SESSION character_set_results = 'utf8mb4'
2021-08-16T10:11:29.130402Z	   20 Query	SET SESSION character_set_results = 'binary'
2021-08-16T10:11:29.130667Z	   20 Query	SELECT COLUMN_NAME,                       JSON_EXTRACT(HISTOGRAM, '$."number-of-buckets-specified"')                FROM information_schema.COLUMN_STATISTICS                WHERE SCHEMA_NAME = 'hunadb' AND TABLE_NAME = 'piyo'
2021-08-16T10:11:29.131010Z	   20 Query	SET SESSION character_set_results = 'utf8mb4'
2021-08-16T10:11:29.131201Z	   20 Query	UNLOCK TABLES
2021-08-16T10:11:29.132686Z	   20 Quit	

---- 多分ここまでがバックアップのためのコード ----

2021-08-16T10:12:17.133740Z	   22 Connect	root@localhost on  using Socket
2021-08-16T10:12:17.134045Z	   22 Query	select @@version_comment limit 1
2021-08-16T10:12:27.437396Z	   22 Query	SELECT DATABASE()
2021-08-16T10:12:27.437700Z	   22 Init DB	hunadb
2021-08-16T10:12:27.438758Z	   22 Query	show databases
2021-08-16T10:12:27.439753Z	   22 Query	show tables
2021-08-16T10:12:27.440834Z	   22 Field List	hiyo 
2021-08-16T10:12:27.442166Z	   22 Field List	piyo 
2021-08-16T10:12:31.924121Z	   22 Query	show tables
2021-08-16T10:12:40.044363Z	   22 Query	truncate table hiyo
2021-08-16T10:12:42.415083Z	   22 Query	show tables
2021-08-16T10:13:24.213758Z	   22 Query	drop table hiyo
2021-08-16T10:13:26.758317Z	   22 Query	show tables
2021-08-16T10:13:30.474725Z	   22 Quit	

---- ここで復元を始めた　---- （ mysql hunadb < dump.sql ）

2021-08-16T10:13:58.825694Z	   23 Connect	root@localhost on hunadb using Socket
2021-08-16T10:13:58.826015Z	   23 Query	select @@version_comment limit 1
2021-08-16T10:13:58.826260Z	   23 Query	/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */
2021-08-16T10:13:58.826450Z	   23 Query	/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */
2021-08-16T10:13:58.826626Z	   23 Query	/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */
2021-08-16T10:13:58.826805Z	   23 Query	/*!50503 SET NAMES utf8mb4 */
2021-08-16T10:13:58.827039Z	   23 Query	/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */
2021-08-16T10:13:58.827220Z	   23 Query	/*!40103 SET TIME_ZONE='+00:00' */
2021-08-16T10:13:58.827404Z	   23 Query	/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */
2021-08-16T10:13:58.827586Z	   23 Query	/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */
2021-08-16T10:13:58.827784Z	   23 Query	/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */
2021-08-16T10:13:58.827967Z	   23 Query	/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */
2021-08-16T10:13:58.828183Z	   23 Query	DROP TABLE IF EXISTS `hiyo`
2021-08-16T10:13:58.832544Z	   23 Query	/*!40101 SET @saved_cs_client     = @@character_set_client */
2021-08-16T10:13:58.832736Z	   23 Query	/*!50503 SET character_set_client = utf8mb4 */
2021-08-16T10:13:58.832935Z	   23 Query	CREATE TABLE `hiyo` (
  `id` int DEFAULT NULL,
  `cost` int DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
2021-08-16T10:13:58.846132Z	   23 Query	/*!40101 SET character_set_client = @saved_cs_client */
2021-08-16T10:13:58.846305Z	   23 Query	LOCK TABLES `hiyo` WRITE
2021-08-16T10:13:58.847102Z	   23 Query	/*!40000 ALTER TABLE `hiyo` DISABLE KEYS */
2021-08-16T10:13:58.848155Z	   23 Query	/*!40000 ALTER TABLE `hiyo` ENABLE KEYS */
2021-08-16T10:13:58.849204Z	   23 Query	UNLOCK TABLES
2021-08-16T10:13:58.849372Z	   23 Query	DROP TABLE IF EXISTS `piyo`
2021-08-16T10:13:58.857241Z	   23 Query	/*!40101 SET @saved_cs_client     = @@character_set_client */
2021-08-16T10:13:58.857442Z	   23 Query	/*!50503 SET character_set_client = utf8mb4 */
2021-08-16T10:13:58.857621Z	   23 Query	CREATE TABLE `piyo` (
  `id` int DEFAULT NULL,
  `cost` int DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
2021-08-16T10:13:58.869241Z	   23 Query	/*!40101 SET character_set_client = @saved_cs_client */
2021-08-16T10:13:58.869410Z	   23 Query	LOCK TABLES `piyo` WRITE
2021-08-16T10:13:58.870143Z	   23 Query	/*!40000 ALTER TABLE `piyo` DISABLE KEYS */
2021-08-16T10:13:58.871175Z	   23 Query	/*!40000 ALTER TABLE `piyo` ENABLE KEYS */
2021-08-16T10:13:58.872237Z	   23 Query	UNLOCK TABLES
2021-08-16T10:13:58.872430Z	   23 Query	/*!40103 SET TIME_ZONE=@OLD_TIME_ZONE */
2021-08-16T10:13:58.872583Z	   23 Query	/*!40101 SET SQL_MODE=@OLD_SQL_MODE */
2021-08-16T10:13:58.872726Z	   23 Query	/*!40014 SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS */
2021-08-16T10:13:58.872864Z	   23 Query	/*!40014 SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS */
2021-08-16T10:13:58.873030Z	   23 Query	/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */
2021-08-16T10:13:58.873174Z	   23 Query	/*!40101 SET CHARACTER_SET_RESULTS=@OLD_CHARACTER_SET_RESULTS */
2021-08-16T10:13:58.873287Z	   23 Query	/*!40101 SET COLLATION_CONNECTION=@OLD_COLLATION_CONNECTION */
2021-08-16T10:13:58.873427Z	   23 Query	/*!40111 SET SQL_NOTES=@OLD_SQL_NOTES */
2021-08-16T10:13:58.873506Z	   23 Quit	

---- ここまでが復元された時のログ ----

2021-08-16T10:14:05.000746Z	   24 Connect	root@localhost on  using Socket
2021-08-16T10:14:05.001258Z	   24 Query	select @@version_comment limit 1
2021-08-16T10:14:12.021449Z	   24 Query	SELECT DATABASE()
2021-08-16T10:14:12.021944Z	   24 Init DB	hunadb
2021-08-16T10:14:12.023879Z	   24 Query	show databases
2021-08-16T10:14:12.025276Z	   24 Query	show tables
2021-08-16T10:14:12.026755Z	   24 Field List	hiyo 
2021-08-16T10:14:12.027480Z	   24 Field List	piyo 
2021-08-16T10:14:13.866961Z	   24 Query	show tables
2021-08-16T10:14:16.344213Z	   24 Quit	
```

ログの見方がよくわからないけど、これのバックアップが始まってそうな部分を見つけて、それ以外の部分のlogに書いてあるコマンドを打っていけばいいのでは？（筋肉で解決）

---- 

## 解説

公式の解説が丁寧なのでそちらを参照。

### 解説に対するメモ

- 'DML' = Data Manipulation Language(select文、insert文など)[参考](https://e-words.jp/w/DML.html) 
- `--base64-output=DECODE-ROWS` はバイナリログが元々ROW形式なので読めるようにするためにつける。
- `-vv` = `--verbose --verbose` (詳細なメッセージを表示) 
- `-- CHANGE MASTER TO MASTER_LOG_FILE='binlog.000018', MASTER_LOG_POS=34626719;` の `binlog.000018` と `34626719`が大事。 
- `binlog.000018` からバックアップ後のデータを復旧する。
- startpositionは`34626719` になる。
- `mysqlbinlog` は MySQLのバイナリログの解析に使われる。

### 試してみた

（私の環境に合わせたコマンドにしてる）

`mysqldump --password=passwordh --opt --single-transaction --master-data=2  hunadb > dump.sql`

--master-data=2が大事だったとは（ちゃんと調べてなきゃ。。）[参考](https://dev.mysql.com/doc/refman/5.6/en/mysqldump.html)

`dump.sql`の`-- CHANGE MASTER TO ` のとこの情報をバックアップ以降の更新の始まりを知るべく確認。

`binlog`もあったのでこれに対して

`mysqlbinlog --no-defaults --base64-output=DECODE-ROWS -vv --start-position=$MASTER_LOG_POS  binlog.000002 | grep -B 10 truncate`
で、出てきた。

truncateを打った時のtimestampがわかったので、
問1は、`select from_unixtime(timestamp);`を打てば良さそう。

`mysqlbinlog --no-defaults --start-position=$MASTER_LOG_POS --stop-position=$timestamp binlog.000018 | mysql -u root -p`
で、復旧の手順は行えることが確認できたと思う。（最初の設定ミスってて、データのバックアップをとってすぐにtrancateしたので確認できなかった😂）

### 感想
全然違くて🥺 DBの授業取ってたのに🥺

## 採点基準

- 問1: 30%
- 問2: 70%