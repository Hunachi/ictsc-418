# 何かがおかしい。。。。

[何かがおかしい。。。。](https://blog.icttoracon.net/2021/03/16/%e4%bd%95%e3%81%8b%e3%81%8c%e3%81%8a%e3%81%8b%e3%81%97%e3%81%84%e3%80%82%e3%80%82%e3%80%82%e3%80%82/)

## 使用環境・ツール
- Go言語
#### ライブラリ
- [Cobra](https://github.com/spf13/cobra)
    - モダンなCLIアプリケーションを簡単に作れるようにしてくれるライブラリ．

## バグの内容

![image](https://blog.icttoracon.net/wp-content/uploads/2021/08/6041e1892f8d9c005ac002d2-512x443.png)

R1にて自作のルーティングアプリケーションを作ったが，そのアプリケーションのミスでそれぞれのHostにpingを飛ばしてもパケットロスを起こしてしまう．

それぞれのHostにpingを飛ばしても、パケットロスを起こしてしまう。
(pingができる時と出来ないときがある。)

## 前提条件
- kernelパラメーターの変更をしてはならない．

## 理想の終了状態
- それぞれのHostにpingを飛ばして、パケットロスが起こらない状態にする．

## その他，公開してくれている情報

ーー起動方法ーー
> cd ~/go/src/github.com/yoneyan/ictsc-ace/cmd/routing
go get .
go build .
sudo ./routing start eth1 eth2 eth3 eth4

（メモ：eth1とかの部分は書き換えてあげる必要がある）

[Source Code](https://drive.google.com/file/d/1njTGorOGDsI1yUYpZlMj0bFLBlbukjX4/view?usp=sharing)

----

## 技術調査

#### 調査方法
ログの仕込み
```go
log.Println("label", "出力させたいもの")
```

#### コードを読んでみる

- CLIアプリケーションを作るためのライブラリとしてCobraが使われてた．
    - [あそんだ](https://github.com/Hunachi/minihuna)
- buildとかはできた
    - NICを用意したりパケットを飛ばし合うのが無理だったので，実際に動かして確認はできなかった．
- コマンドとしてあるもの
    - routor init
    - routor start ←今回叩いているコマンド
    - routor recieve
    - 未実装
        - routor init
        - routor store
        - routor router

- ファイル別の実装されている内容
    - cmd/routing/routing.go
        - メイン関数が書いてあるだけ．
    - pkg/routing/arp/*
        - Macアドレスを取得するためのコード（arp.go）とMacアドレスの情報を入れるための構造体（interface.go）
    - pkg/routing/cmd/*
        - CLIのコマンドの実装
    - pkg/routing/route/*
        - 今回使われていない
    - pkg/routing/router/*
        - ルーティングの処理が書かれている部（router.go）とルーティング情報を詰めるための構造体（interface.go）
    - pkg/routing/tool/*
        - 今回使われていない
    - pkg/routing/interface.go
        - 今回使われていない


## 考えられる検証手順

- コードを読む（読んだ）
- Logを埋め込む
- どんなパケットが落ちてるかみる

## 原因の候補

- コードを読んだ感想
    - Ipv6に対応してないから？
        - これだったらトラコンじゃない．
    - MacアドレスとIPアドレスの組みが変わったときに対応できないから？

---- 

## 解説

### 原因

> 原因はGolangにてルーティングの実装に問題があることが原因です。
Golangにて、Channelを使った処理にミスが生じており、適切なNICにパケットを送り出せていないため起きている問題です。

> HostA => HostD宛にPingを飛ばした場合でも、Channelの実装ミスによりHostA,B,C,Dのどれかにパケット転送してしまうというバグによって引き起こされる問題です。

### 解決方法

```go
pkg/routing/router/router.go
 
 
 func routerReceive(device string) error {
                                if err != nil {
                                        log.Println(err)
                                }
+                       } else {
+                               packets <- msg
                        }
                }
        }()

```


### 解説に対するメモ

`router.go`内の，
```go
msg := <-packets
			if msg.Interface == mac.String() {
				err := handle.WritePacketData(msg.Packets)
				if err != nil {
					log.Println(err)
				}
			}
```
の部分で，あるスレッド（goroutine）が指定したネットワークデバイス(NIC)で送信しないパケットを受け取った場合にパケットを破棄してしまっていることが原因．
本当は，自分が送信するパケットじゃなかったら他のNICを指定したスレッド（goroutine）に渡してあげないといけない．
と言うことに気が付けなくてﾋﾟｴﾝ🥺

## 採点基準

パケットロスをせずにpingが飛ぶこと(100%)