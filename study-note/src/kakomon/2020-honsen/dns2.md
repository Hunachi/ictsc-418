# DNSサーバを作りたかったらしい
[問題リンク](https://blog.icttoracon.net/2021/03/16/dns%e3%82%b5%e3%83%bc%e3%83%90%e3%82%92%e4%bd%9c%e3%82%8a%e3%81%9f%e3%81%8b%e3%81%a3%e3%81%9f%e3%82%89%e3%81%97%e3%81%84/)  

## 概要
あなたは同僚から助けを求められた。彼は社内のDNSサーバの構築ログに基づいて環境構築を試みたが、テストとして実行したコマンドでは期待していた出力が行われなかったらしい。原因を調査して、エラーを解決してあげよう。  

## 前提条件
- ns01はmaster、ns02はslaveサーバとして機能させたい
- トラブルに関係しない要素については変更しない
- ns01,ns02についての接続情報が与えられている

## 初期状態
ns02で```dig @localhost red.prob.final.ictsc.net```が解決できない

## 終了状態
- ns02で```dig @localhost red.prob.final.ictsc.net```が解決できる
- 原因が特定されて報告される  
    - トラブル解決前に期待されていた動作をしている

## 考えられる原因・解決方法
### digコマンドについて
[参考Qiita](https://qiita.com/hana_shin/items/e99f64a01f2632b7a719)
digコマンドは、DNSサーバに対して問い合わせを行い、その結果を表示する。  

#### ちなみに
DNSサーバにはキャッシュDNSと権威DNSの二種類がある。
権威DNSにアクセスが集中しないよう、しばらくの間はキャッシュDNSサーバが権威DNSから受け取った応答を保持して、その内容を使ってクライアントに応答する。  


digコマンドの話に戻るが、digコマンドはBIND 9に付属するコマンドである。  
nslookupと機能が似ているが、digコマンドによって得られる情報量が格段に多い。  

### BIND 9とは
Berkley Internet Name Domainの略で、世界中で最も多く利用されているDNSサーバのこと。  
他にもDNS機能を提供するソフトウェアは存在するが、BINDを使うサーバがほとんどである。  

[DNSサーバの構築トラブルシューティング](https://dnsops.jp/event/20140626/dns-beginners-guide2014-mizuno.pdf)  
本問題のdigの実行結果がわからないので、考えられる要因が多岐にわたる

## 解法
今回の問題文に、『彼は社内のDNSサーバの**構築ログ**に基づいて環境構築を試みたが、テストとして実行したコマンドでは期待していた出力が行われなかった』とあるので、ログを見る必要がある。  
どのログを見るかについては、コマンドを起動するための設定ファイルに関するものを見る必要がありそう。  
例えば、今回であればdigコマンドがうまくいっておらず、digコマンドの設定ファイルはBINDのものなので、
```systemctl status bind9```とする。  

![ログ表示結果](https://user-images.githubusercontent.com/46861288/130339607-836f732b-90d4-4f0e-8990-6784770abf82.png)

このログを見ると、BINDが設定ファイルのエラーによって起動していないことがわかる。  
```/etc/bind/named.conf.prob```についてエラーが出ているので、この設定部分を変更する必要がある。  

### viewとzoneとは
[参考リンク](https://ygg-tech.com/2020/09/bind_view/)  

viewは、一言で言ってしまえば「接続条件によってDNSクエリ結果を変える機能」である。  
Bindの Ver.8 から使用可能であり、CentOS7であれば Ver.9 なので問題なく利用可能。
viewステートメントによって、アクセス元のIPアドレスに応じて、検索クエリに応答するゾーン情報を分けることができる。  
活用する状況としては以下が挙げられる。  

- 社内にWebサーバがあり、複数のLANに足を出している
- 各LANのクライアントには、自身が所属するLANのWebサーバのIPを返したい
- DNSは1台で済ませたい

![view説明画像](https://user-images.githubusercontent.com/46861288/130374774-e48a9a6b-f68a-4b72-8354-010abdff9b6b.png)

この画像の中では、
- Secure LANに所属するクライアントには192.168.0.1を返す
- Remote Access LANに所属するクライアントには、172.24.0.1を返す
というフローが実現できる。  

設定ファイルの書き方は[参考リンク](https://ygg-tech.com/2020/09/bind_view/)参照。  

ちなみに、zoneをドメイン名や単一DNSサーバと関連づけることは間違いである。  
DNSゾーンは複数のサブドメインを含むことができ、複数のゾーンが同じサーバに存在できる。  
ゾーンファイルに、DNSサーバに保存されているゾーン内の全てのドメインの全てのレコードが含まれ、記載される。

### 解法に戻る
viewの説明箇所で記述したように、複数のviewを使う際に、同じ名前がついた実態は異なるzoneを含んでいても、クライアント毎に異なるレスポンスを返すことが可能である。  
しかし時には、複数のviewが同一のzoneを持つことが望まれる場合もある。  
そこで用いられるのが ```in-view``` zone オプションである。 
[in-view公式解説](https://bind9.readthedocs.io/en/latest/reference.html?highlight=in-view#multiple-views)  
これは一言で言うと、viewをまたいだゾーンの共有である。   
```in-view``` オプションによって、事前に設定されたviewの中で定義されたzoneを、viewが参照することができるようになる。  
これは設定ファイルにおいて後の部分に記述されているviewを参照することはできない。  

2つ以上のviewで同じzone(type slave)を設定するとエラーになるので、2回目以降に使用する際にはin-viewオプションを使うとエラーが回避できる。  

### BINDの設定方法
[DNSサーバ構築手順参考リンク](https://go-journey.club/archives/395)  
全体設定ファイルnamed.confは、デフォルトでは/etc/named.confに配置されている。   

ちなみに、BINDをchrootしておくと、bind サービスは /var/named/chroot より上のディレクトリには行けなくなり、仮に bind を乗っ取られても他のディレクトリには被害が及ばない。  
つまり、セキュリティを確保することができる。  

chrootの場合には、自動的に```/var/named/chroot/etc/named```配下に named.conf などの設定ファイルができている。

```systemctl restart named```として再起動ができる。  
ちなみに、named-chrootをインストールした場合、
- named　→　自動起動　disable
- named-chroot　→　自動起動　enable

の設定をしないと、namedが二つ起動している状態になってしまうので注意が必要である。