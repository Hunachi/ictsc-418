# jsonが壊れた！！

[jsonが壊れた！！](https://blog.icttoracon.net/2020/11/02/json%e3%81%8c%e5%a3%8a%e3%82%8c%e3%81%9f%ef%bc%81%ef%bc%81/)

## 使用環境・ツール
- Go
- json

## 問題文でされた操作
GoでAPIサーバを作った。

## バグの内容
json文字列をparseするときにデータがlossしてしまう。

## 初期状態

VM名 gogo には、golang 1.14.6がインストールされている。
VMのホームディレクトリには`testcode.go` が置かれている。

`$ go run testcode.go` を実行すると以下の結果が得られる。

- 実行結果

```
{10 Gopher 0}
```

## 理想の終了状態
コマンド `$ go run testcode.go` を実行すると

```
{10 Gopher passwordnohash 99}
```

上記の結果が得られる。

また、問題の解決が永続化されている。

----



## 考えられる検証、修正手順

### バグの原因を特定する案

- testcode.goの問題
  - 構造体とJSONの構造をそろえて定義してあるか確認する
    - [qiita](https://qiita.com/nayuneko/items/2ec20ba69804e8bf7ca3)
  - nullがjsonに含まれていたときのことを考慮しているか確認する
    - バージョンによって、nullがjsonに含まれていると扱い方によってはエラーになる
    - [はてブ](https://y0m0r.hateblo.jp/entry/20131228/1388189124)
  - 文字コード周りを確認する
    - jsonの仕様的に文字列の中に`'\n'`を含んではいけない
    - [はてブ](https://blog.nishimu.land/entry/2014/04/16/213243)
  - floatとint周りを確認
    - GoはJSONの中に含まれる数値を `float64` 型として扱う
    - [qiita](https://qiita.com/tutuz/items/fedb8e3a1137d046f418#json%E6%95%B0%E5%80%A4%E3%81%AE%E3%82%A4%E3%83%B3%E3%82%BF%E3%83%BC%E3%83%95%E3%82%A7%E3%83%BC%E3%82%B9%E5%9E%8B%E3%81%B8%E3%81%AE%E3%83%87%E3%82%B3%E3%83%BC%E3%83%89)
- jsonの問題
  - 形式がおかしくないか確認する

### バグの原因が見つかった際の手順

- testcode.goの問題
  - testcode.goを修正
- jsonの問題
  - jsonを修正

### 修正手順案

- エラーメッセージを読むと、testcode.goの問題のポツで書いた4つの原因のどれか、またはそれ以外かが分かる

- もしエラーメッセージに情報が無ければ、ひとまずtestcode.goを確認した後、jsonを地道に確認

----



## 解説

### 原因

この問題はUserという構造体のtag指定が2点間違っていることが原因でした。

コンパイルでは見つけられない。

### 1

=を:に修正する。

------

**変更前**

```
json=""
```

**変更後**

```
json:""
```

------

**実行結果**

```
{10 Gopher passwordnohash 0}
```

------

### 2

“が抜けているため足す。

------

**変更前**

"access_count`

**変更後**

"access_count"`

------

**・2だけを修正した場合**

```
{10 Gopher 0}
```

**・1と2を修正した場合**

```
{10 Gopher passwordnohash 99}
```

### 解法２

`go vet` コマンドによる静的解析

- go vetとは？
  - `本当に問題になるかは分からないし、ビルドは通せるんだけど、こういう懸念があるかもね` 的な問題を
    コンパイラとは別の観点でチェックしサジェストをしてくれる、賢い子
  - [qiita](https://qiita.com/marnie_ms4/items/b343165efb4235906db7)