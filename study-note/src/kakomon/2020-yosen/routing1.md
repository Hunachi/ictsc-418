# 経路が受け取れない！
解いた人:[nonnonno](https://github.com/nonnonno)

参照した問題・解説のサイト:[経路が受け取れない！](https://blog.icttoracon.net/2020/11/02/%e7%b5%8c%e8%b7%af%e3%82%92%e5%8f%97%e3%81%91%e5%8f%96%e3%82%8c%e3%81%aa%e3%81%84/) 

## 使用環境・ツール
- ルータ
- BGP (経路制御プロトコル)

## 問題文でされた操作
[経路が受け取れない！](https://blog.icttoracon.net/2020/11/02/%e7%b5%8c%e8%b7%af%e3%82%92%e5%8f%97%e3%81%91%e5%8f%96%e3%82%8c%e3%81%aa%e3%81%84/)の一枚目の画像を参照

AS65000, AS65001, AS65002に所属するルータを3つ作成し、これらのルータ全てからフルルートをAS65010宛に流した。  


## バグの内容
なぜかAS65010に所属するルータで経路を受け取れない


## 理想の終了状態
AS65010に所属するルータにおいて、3つのルータ全てからのフルルートを受信したい  


## そもそもBGPとは
Border Gateway Protocolの略称で、パケットの宛先を正確に把握し、維持していくための経路制御を行うための代表的なプロトコル。  
BGPは、経路制御を行う組織ごとにインターネットの世界で唯一の番号が割り当てられ、個々の経路を識別するという特徴があるため、組織内での設計と機器の準備だけでは利用することができない。  
その唯一の番号はAS (Autonomousu System) 番号と呼ばれ、IANA (Internet Assigned Numbers Authority) が管理する。  
AS番号とIPアドレスの割り当てを受けた後、上流ISPや経路情報を持つ相互接続点であるIXに接続が必要である。  
その後に、自組織のBGPルータと接続先ルータ間でピアと呼ばれる経路交換を行う設定を行う。  
BGPの経路制御はこのピアを通して行われる。
ピア設定後、自組織のIPアドレスを接続先へピアを通して通知する必要があり、このことを「アドレスをアナウンスする」と言う。 
ちなみに、アナウンスだけでなく、上流ISPからインターネット上の経路を取得することも必要。
BGPルータはピアを確立するために、メッセージ形式で機能情報を交換し続ける。  

なお、BGPはマルチキャストで動的にネイバーを自動検出することはできず、自分のAS内のネットワークをBGPルートとしてアドバタイズするときはnetworkコマンドを使用する。  
networkコマンドで通知するためには、前提としてそのルートをルーティングテーブルで学習していることが必要。  
自分のAS内のネットワークを通知する必要がない場合には、networkコマンドの設定は不要。  

## 考えられる検証、修正手順

- ハードが正常に動いていない
    - dmesgコマンドなどでハードウェアの動作を確認する
    - BGPのデーモン自体が動いていない可能性がある
    [BGP有効化など設定](https://www.infraexpert.com/study/bgpz06.html)

- 経路情報が間違っており、通信ができていない/ルーティングテーブルが間違っている
    - [経路情報確認コマンドリンク](https://engineers-life.com/linux_command/network_linux/linux_ip-route/)

- その他トラブルシューティングの原因切り分け手順は以下のリンク

[BGPトラブルシューティングフローチャート、Cisco](https://www.cisco.com/c/ja_jp/support/docs/ip/border-gateway-protocol-bgp/22166-bgp-trouble-main.html)

[BGPルートがアドバタイズされない場合のトラブルシューティング、Cisco](https://www.cisco.com/c/ja_jp/support/docs/ip/border-gateway-protocol-bgp/19345-bgp-noad.html)


[BGPの設定・確認参考、Juniper webページ](https://www.juniper.net/documentation/jp/ja/software/junos/bgp/topics/topic-map/troubleshooting-bgp-sessions.html)

## 解説
### 解法

- /var/log/messageを確認し、ルータのエラーメッセージが表示されていることを確認
- vyos(ルータなどの機器が利用するネットワークOS)によるBGPプロセス全体が停止していることを
    - ```$ show ip route```でBGPが有効ならば、対向ルータとのBGP stateがEstablishedであり、広報されている経路情報を新しく受け取っていることがわかる
    - [VyosによるBGP設定、ISP構築手順のコマンド参考、朝日ネット](https://techblog.asahi-net.co.jp/entry/2018/08/17/161853)
- (今回はBGPが無効になっており、)外部からあまりにも経路広告が行われ、メモリがオーバーフローしていることを認識(予備知識?)

#### 得点ポイント
1. BGPデーモンが停止している
2. 経路が多すぎてメモリがオーバーフローしている

