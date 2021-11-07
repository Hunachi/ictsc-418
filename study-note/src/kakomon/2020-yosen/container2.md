# ダイエットしようぜ
解いた人:[とり](https://twitter.com/lnamikol)

参照した問題・解説のサイト:[問題文](https://blog.icttoracon.net/2020/11/02/%e3%83%80%e3%82%a4%e3%82%a8%e3%83%83%e3%83%88%e3%81%97%e3%82%88%e3%81%86%e3%81%9c%ef%bc%81/)

## 使用環境・ツール
- Docker
- Go


## 問題文でされた操作
- GoでSHA256する`hash.go`を作成し、コンテナが欲しかったためDockerfileを作成
- ビルドしてイメージを作成した

## バグの内容
- 作ったDockerImageが大きすぎてテンションが上がらない。
- ヒント: `あなたには、Dockerfileを編集したりビルドコマンドを変えたり、あるいはビルド後のイメージに対してなにかしたりしてイメージを小さくしてほしい。`

## 条件
- `$ docker images`で当該イメージを見ると796MBである。
- Dockerのコマンドはsudoなしで実行できる。
- hash.goを編集してはならない。
- もしよければ`$ make build`でDocker imageを作って欲しい。(任意)

## 理想の終了状態
- `$ docker images`で当該イメージを見ると796MBより小さくなっている
    - 可能な限り小さくしてほしい
----

## 考えられる原因
- `Dockerfile`に必要ないものを記述している
- 軽量化を頑張っていない

### 解決方法
- Multi-stage builds
- Alpine Linux
- 不要なパッケージやファイルの削除
https://qiita.com/ytanaka3/items/8c308db2ee58ea63626a

参考: [YouTube](https://www.youtube.com/watch?v=wGz_cbtCiEA)
- Small Base image
    - デフォルトで軽量化しているimageを提供している場合がある
        - node.jsならslimバージョンが存在する(Docker側から公式で)
    - 存在しない場合は`Alpine Linux`をスタートにしてコンテナを作る
        - Alpine Linux?
- Builder Pattern
    - インタープリター -> 直接コードを実行する
    - コンパイラ言語 -> コンパイルして実行する
        - コンパイラした時のツールは実際のDocker imageを実行する際には必要ない

重いやつ
```
FROM go:onbuild
EXPOSE 8080
```

軽めにしたやつ
```
FROM golang:alpine
WORKDIR /app
ADD . /app
RUN cd /app && go build -o goapp
EXPOSE 8080
ENTRYPOINT ./goapp
```
変更した点:
- オンビルドイメージからAlpine Linuxに変更
    - いつも見るやつだ！！！

さらに軽く変更
```
FROM golang:alpine AD build-env
WORKDIR /app
ADD . /app
RUN cd /app && go build -o goapp

FROM alpine
RUN apk update && apk add ca-certificates && rm -rf /var/cache/apk/*
WORKDIR /app
COPY --from=build-env .app/goapp /app

EXPOSE 8080
ENTRYPOINT ./goapp
```
変更した点:
- 未処理のimegeをFROM alpineで使った
- Alpine LinuxではSSL認証が入ってなくてHTTPSが失敗するため、ルートCA認証をインストールする
- copyでコンパイル済みのコードを２こ目のコンテナにコピーする

**これをマルチステージビルド**というらしい！
700MB -> 12MBになる。わんちゃん問題はこれ参考にした説までもあるな🤔

---- 

## 解説
### 解法
- ベースイメージをalpineベースのものに変える(312MB)
- docker-slimを使う(57.9MB)
- マルチステージビルドを使う(8.14MB)

#### 得点
- (ベースイメージをAlpineにして)サイズを400MB以下にした: 40%
- (Docker-slimを使って)サイズを100MB以下にした: 70%
- (Multi Stage Buildを使って)サイズを10MB以下にした: 100%

Multi Stage Buildをすべきみたいだ

### 解決方法（もっと詳しい説明を書き加える）
#### docker-slimを使う
- 導入方法
```
$ curl -L -O https://github.com/docker-slim/docker-slim/releases/download/1.22/dist_linux.tar.gz
$ tar zxvf dist_linux.tar.gz
$ cd dist_linux/
$ ls
docker-slim  docker-slim-sensor
# 多分pathを通す。まあ通さなくても実行できるが
$ docker-slim [version|info|build|profile] [--http-probe|--remove-file-artifacts] <IMAGE_ID_OR_NAME>
```

https://qiita.com/ryuichi1208/items/c96d39a57e11d54f02bf

#### マルチステージビルドを使う
さっき説明した感じのやつ
```
FROM golang:1.11
WORKDIR /work
COPY ./hash.go /work
RUN go build -o app hash.go
 
FROM alpine:latest # FROM scratchでもいいっぽい。
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=0 /work/app .
CMD ["./app"]
```
最初の4行でビルドし、次の5行でビルドした実行ファイルのみalpineにcopyするスタイル。

#### その他
- `go build -ldflags '-s -w'` を使うと実行ファイルが小さくなるらしい
    - `-ldflags '[pattern=]arg list'`
        - arguments to pass on each go tool link invocation.(ツールリンクの起動時に渡される引数です。)
    - DWARF とシンボルテーブルの情報をバイナリに残さずコンパイルできる
        - https://qiita.com/kitsuyui/items/d03a9de90330d8c275c8
    - `-ldflags="-w"` でDWARFのシンボルテーブルを生成しないようにする
    - `-ldflags="-s"`によってデバッグのために使われるシンボルテーブルを全部生成しないようにする
        - http://tdoc.info/blog/2016/03/01/go_diet.html
        - このサイトにはUPXの話も載っている
    
        
- UPX圧縮
    - ちょっとだけ(1MB)圧縮される
    - バイナリは半分くらいになる
    - コマンドでできるっぽい
    - https://future-architect.github.io/articles/20210520b/

