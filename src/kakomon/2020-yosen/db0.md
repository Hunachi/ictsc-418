# 備品は何処へ
解いた人:[Hunachi](https://twitter.com/_hunachi)

参照した問題・解説のサイト:[備品は何処へ](https://blog.icttoracon.net/2020/11/02/%e5%82%99%e5%93%81%e3%81%af%e4%bd%95%e5%87%a6%e3%81%b8/)

## 使用環境・ツール
- MySQL

## 初期状態
- MySQLが起動しroot権限にて正常動作することを確認している
- 使用するデータベースはListである
- データベースListのテーブルには以下がある
    - Equipment_list(備品リスト)
    - Order_company_list(注文会社リスト)
    - Manufacturing_company_list(製造会社リスト)
    - Location_list(所在地リスト)

## 終了状態
- 提出されたCSVデータは値段(priceカラム)の高い順位にソートされている
- 提出時に付随するカラムは以下の通り
    - Equipment_list.ID
    - Equipment_list.Name
    - Order_company_list.Name
    - Manufacturing_company_list.Name
    - Equipment_list.Price

----

## 技術調査

### MySQLの使い方

#### CSVを生成する方法

``` sql
SELECT * FROM {DB_NAME} INFO OUTFILE 'hoge.csv'
```
` INTO OUTFILE 'hoge.csv'`のところで出力するファイルを指定している．

参考：https://qiita.com/catatsuy/items/9fdf4423d5f4885b9bf9
csvからデータのimportをする方法も書いてるサイト：
https://proengineer.internous.co.jp/content/columnfeature/6776#section100

#### テーブルの結合方法

##### 外部結合
条件に一致するもの同士を結合させるのに加えて，どちらかのテーブルにしか存在しないレコードも残して結合する． https://zenn.dev/naoki_mochizuki/articles/60603b2cdc273cd51c59

`LEFT OUTER JOIN`の場合は，先に書いたテーブル側のカラムが消えない．
`RIGHT OUTER JOIN`の場合は，後に書いたテーブルのカラムが消えない．

```
SELECT * FROM hoge
    LEFT OUTER JOIN piyo ON hoge.id = piyo.id;
```

↑hogeのうち，どのpiyo.idとも一致しないhoge.idのレコードも残されてpiyoの方の値はnullになる．どのhoge.idとも一致しないレコードは消される．

##### 内部結合

テーブルの指定したカラムが一致するものだけを結合する方法．
条件に合う相手がいないレコードは消される． https://zenn.dev/naoki_mochizuki/articles/60603b2cdc273cd51c59

```
SELECT * FROM hoge
    INNER JOIN piyo ON hoge.id = piyo.id;
```

##### 自然結合

結合条件として，同じ名前のカラム同士で結合する場合，`NATURAL`を付けることで結合条件を省略することができる方法． https://www.dbonline.jp/sqlite/join/index4.html

```
SELECT * FROM hoge
    NATURAL INNER JOIN piyo ON hoge.id = piyo.id;
```

##### 複数テーブルの結合方法

```
SELECT * FROM hoge
    INNER JOIN piyo ON hoge.id = piyo.id
    INNER JOIN fuga ON piyo.f_id = fuga.id;
```

のように書けばいい．

## 解決方法
コーディングテストのようにいい感じのSELECT文を書けば良さそう．（トラブルシューティングではなさそう）

### STEP 0.
DB(List)に接続する．

- `sudo mysql`
- `mysql> use List;`

一応テーブルの構造も確認する

- `mysql> desc List;`

### STEP 1.
各テーブルのカラムを確認するために，それぞれに対して
`mysql> SELECT * FROM {DB_NAME}` をする．

IDの指定が被っている部分を確認して結合する時の条件を考える．

### STEP 2.
SELECT文をかく．

0. 必要なカラムを出力するようにする

``` sql
SELECT e.id, e.name, o.name, m.name, e.price
```

1. テーブルの結合を行う
どういうデータが入ってるのかよくわからないので，内部結合で行うべきか外部結合で行うべきかわからないぞ．(内部結合で書いてみるぞい．)

``` sql
FROM Equipment_list as e 
INNER JOIN Order_company_list as o ON {条件} 
INNER JOIN Manufacturing_company_list as m ON {条件}
INNER JOIN Location_list as l ON {条件}
```

2. 並べ替える条件を入れる

```
ORDER BY e.price DESC
```

3. CSVにするように

``` sql
INTO OUTFILE 'submit.csv'
```

4. 今までのを結合 

``` sql
SELECT e.id, e.name, o.name, m.name, e.price
INTO OUTFILE 'submit.csv'
FROM Equipment_list as e 
INNER JOIN Order_company_list as o ON {条件} 
INNER JOIN Manufacturing_company_list as m ON {条件}
INNER JOIN Location_list as l ON {条件}
ORDER BY e.price DESC;
```

---- 

## 解説

複数のテーブルから必要な情報を結合し情報を出力できるかという問題です．
余分なデータを表示させたくないため内部結合を使います．

`Equipment_list`のIDを各テーブルのIDと結合させ表示、そして条件に合わせて検索、ソートを行うという問題でした。

### 答え

``` sql
select Equipment_list.ID,Equipment_list.Name,Order_company_list.Name,Manufacturing_company_list.Name,Equipment_list.Date,Equipment_list.Price  from Equipment_list 
inner join Order_company_list on Equipment_list.Order_company= Order_company_list.ID
inner join Manufacturing_company_list on Equipment_list.Manufacturing_campany = Manufacturing_company_list.ID
inner join Location_list on Equipment_list.Use_place = Location_list.ID
where Manufacturing_company_list.ID=5 and Location_list.ID<=5
order by Price DESC
```

で確認した後，

``` sql 
$(sudo) mysql -u root -p -D List -e " select Equipment_list.ID,Equipment_list.Name,Order_company_list.Name,Manufacturing_company_list.Name,Equipment_list.Price  from Equipment_list inner join Order_company_list on Equipment_list.Order_company= Order_company_list.ID inner join Manufacturing_company_list on Equipment_list.Manufacturing_company = Manufacturing_company_list.ID inner join Location_list on Equipment_list.Use_place = Location_list.ID where Manufacturing_company_list.ID=5 and Location_list.ID<=5 order by Price DESC;" |sed 's/\t/,/g'> re.csv
```

CSV形式で出力↑．

``` sql
$cat re.csv
ID,Name,Name,Name,Price
24,pc_x,Order_company_E,Manufacturing_company_E,1600
17,pc_q,Order_company_J,Manufacturing_company_E,800
```

### 解説に対するメモ

コマンドラインのオプションたち
参考：https://qiita.com/yulily@github/items/54cb6ccaacf39977455c

自分が，`特定の製造会社のパソコン（Manufacturing_company_E）、特定の建物（HQ）で相性が悪く交換するためリストを作成してほしい`という条件をつけろという部分の意味が理解できてなくて， `where Manufacturing_company_list.ID=5 and Location_list.ID<=5` をつけ忘れたことが判明した．

#### 本来すべきだったと思われる追加の作業

特定の条件を見つける．

```sql
SELECT * FROM Manufacturing_company_list　WHERE Manufacturing_company_list.Name = 'Manufacturing_company_E';
```

```sql
SELECT * FROM Location_list　WHERE Location_list.Name = 'HQ';
```

これらの結果から，以下を見出す必要があった．

```sql
where Manufacturing_company_list.ID=5 and Location_list.ID<=5
```

## 採点基準
- 上記のカラムが表示できるSQL文を作成できている。[40%]
- 正しく出力できている。[20%]
- 売り上げ１位から順表示ができている。[40%]