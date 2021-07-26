# またビルド失敗しちゃった～…

[またビルド失敗しちゃった～…](https://blog.icttoracon.net/2020/11/02/%e3%81%be%e3%81%9f%e3%83%93%e3%83%ab%e3%83%89%e5%a4%b1%e6%95%97%e3%81%97%e3%81%a1%e3%82%83%e3%81%a3%e3%81%9f%ef%bd%9e/)

## 使用環境・ツール
- Docker
    - Dockerfile
- Go

## バグの内容
Dockerのマルチステージビルドを使って、Goのバイナリをコンテナ上で実行しようとしたらうまく立ち上がらない。Dockerfileが間違えている。

`~/app/Dockerfile`を用いて`docker image build -t ictsc2020:0.1 .`したあとに、`docker run -p 80:1323 [コンテナID]`をするとエラーが表示され、コンテナ上のバイナリが正常に実行できない。

## 理想の終了状態
`curl localhost`で`Welcome to ICTSC2020!`が返ってくる。
また、問題の解決が永続化されている。

----

## 技術調査
#### Dockerのマルチステージビルドとは？
- https://docs.docker.com/develop/develop-images/multistage-build/
    - ↑の訳らしい: https://qiita.com/carimatics/items/01663d32bf9983cfbcfe
- 必要なバイナリorファイルだけをコピーしてきたイメージを楽に作ることができるようになる．

#### Dockerfileの内容をみてみる
```
// ~/app/Dockerfileの内容

1:FROM golang:1.15.0 AS builder
2:ENV GO111MODULE=on
3:ENV GOPATH=
4:COPY ./server/main.go ./
5:RUN go mod init ictsc2020
6:RUN go build -o /app ./main.go
 
 
7:FROM alpine:3.12
8:COPY --from=builder /app .
9:EXPOSE 1323
10:ENTRYPOINT ["./app"]

```

- 1行目: 新しい（１つ目の）ビルドステージの開始
    - 1つ目のコンテナの作成
    - golang:1.15.0を親イメージとして指定
    - ビルドステージに builder という名前をつけている．（デフォルトでは 0 から連番で付けられる．）
- 2行目：　Go modulesを使うという指定
    - 何も指定しないor auto指定だとgo.modファイルがある時だけGo modulesを使うとみなされる．
    - Go modulesを使うと，importしたいライブラリと，ライブラリの依存ファイルをgo.modに書いてあげるとダウンロードしてくれてビルドしてくれるようになる．
- 3行目:Goのファイルたちが置いてあるパスの指定
    - 特に指定してない．
    - import文の解決に使うために設定するらしい．
- 4行目:./server/main.go を ./以下にコピー
- 5行目：go moduleの作成
- 6行目:main.goをビルドしてappバイナリファイルを生成する

- 7行目: 新しい（2つ目の）ビルドステージの開始
    - 2つ目のコンテナの作成
    - alpine:3.12を親イメージとして指定
    - https://alpinelinux.org/
- ８行目：　builderという名前のビルドステージ（１つ目に作ったもの）の/appを.以下に持ってくる
- 9行目: 1323番ポートを公開する
    - https://zenn.dev/suiudou/articles/5e1dfd1008bf29
- １０行目：　./appを必ず実行する
    - https://pocketstudio.net/2020/01/31/cmd-and-entrypoint/

## 考えられる検証、修正手順

- エラーの内容を確認する
- main.goのコードがおかしい
    - main.goのコードを確認してみる
-   `docker multistage build golang runtime error`でググって一番に出てきたやつ：https://stackoverflow.com/questions/56057688/issue-with-docker-multi-stage-builds
    -  解決手順
        -  `ldd`コマンドを使ってバイナリファイルの依存関係の確認
        -  `RUN`コマンドに`CGO_ENABLED=0`を付け加えて実行し直してみる



#### 原因ではなさそうなこと
- imageは作成できており実行時エラーのため,Dockerファイル内のコマンドが途中で転けているとかではない
- ポートの指定方法

---- 

## 解説

### 前提
> golangで書いたプログラムを`go build`すると、基本的には静的リンクになるが`net`パッケージを使用していた場合、自動的に動的リンクになる。(go1.4から)

### 原因
Dockerのマルチステージビルドをする際に、ビルドされたバイナリが動的リンクなせいでコンテナ間を移動させたあとに実行しようしてもうまくいかなかったため．

### 解決方法

1. `~/app/Dockerfile`の6行目の`RUN go build -o /app ./main.go`を`RUN CGO_ENABLED=0 go build -o /app ./main.go`に変更
2. コンテナをbuildし直す．　`docker image build -t ictsc2020:0.1 `
3. 実行し直す `docker run -p 80:1323 [コンテナID]`

#### 想定外解放としてあったもの
- 動的リンクのまま、lddコマンドを使用し依存しているライブラリを特定して、そのライブラリをbuilderコンテナから実行用のコンテナにコピー・配置する

### 解説に対するメモ
- 実行時に`Not Found`っていうエラーが起きた時は，動的リンクがうまくいっていないパターンも考えるといいっぽい．[動的リンク](https://e-words.jp/w/%E5%8B%95%E7%9A%84%E3%83%AA%E3%83%B3%E3%82%AF.html#:~:text=%E5%8B%95%E7%9A%84%E3%83%AA%E3%83%B3%E3%82%AF%E3%81%A8%E3%81%AF,%E3%81%97%E3%81%A6%E8%B5%B7%E5%8B%95%E3%81%99%E3%82%8B%E3%81%93%E3%81%A8%E3%80%82)

- バイナリを静的にコンパイルさせる為には`CGO_ENABLED=0`を付ける．
    - https://www.docker.com/blog/containerize-your-go-developer-environment-part-1/

- `ldd`コマンド
    - すべての依存関係のパス名をリストで表示してくれる．
    - 動的リンクやファイルの依存関係のエラーの時に役に立ちそう？
    - 多分 `ldd app` で調べることができる．
    - https://www.ibm.com/docs/ja/ssw_aix_71/l_commands/ldd.html
    - https://wa3.i-3-i.info/word13813.html

## 採点基準
1. Dockerfileに正しい改善がなされている: 50%
2. curl localhostをして、正しいレスポンスが返ってくる: 50%