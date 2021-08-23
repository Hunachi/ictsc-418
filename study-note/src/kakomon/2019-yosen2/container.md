# 君k8s得意って言っていたよね？
Edit by 鳥山

[君k8s得意って言っていたよね？](https://blog.icttoracon.net/2019/12/10/ictsc2019-%e4%ba%8c%e6%ac%a1%e4%ba%88%e9%81%b8-%e5%95%8f%e9%a1%8c%e8%a7%a3%e8%aa%ac-%e5%90%9bk8s%e5%be%97%e6%84%8f%e3%81%a3%e3%81%a6%e8%a8%80%e3%81%a3%e3%81%a6%e3%81%84%e3%81%9f%e3%82%88%e3%81%ad/)

## 使用環境・ツール
- Kubernetes
- Redmine
- MariaDB

### Redmine
プロジェクト管理ができるオープンソースソフトウェア。Dockerに公式イメージが存在する。
[Redmine](https://redmine.jp/overview/)

### MariaDB
MariaDBは、MySQL派生として開発されている、オープンソースの関係データベース管理システムである。RDBMS。

https://mariadb.org/

## 問題文でされた操作
- Redmineは指定のManifest(Redmine_Manifest)でデプロイしてください。
- Redmine_Manifestは変更出来ません。
- Redmine_Manifest内のコンテナイメージはcontainer-registryから取得してください。
- マニフェストの再適用, OSの再起動の操作は可能です。
- 誤操作等で競技続行不可の場合は出題時環境への復元のみ承ります。
Kubernetes上にRedmineサービスを稼働させる問題です。
出題時にはRedmineを構成するRedmine-Pod, MariaDB-PodがPendingとなっており、利用不可の状態です。
コンテナが稼働しない原因を突き止め対処することでRedmineサービスを稼働させることができます。

問題解決のために以下の原因を解決する必要があります。

1. masterへpodのデプロイに関するtaints(テインツ)の削除
2. コンテナランタイムcri-oにinsecure-registryの設定を追加
3. MariaDBのPersistentVolumeのディレクトリ権限(Permission)を修正


## 理想の終了状態
- VNCクライアントのブラウザからRedmineが閲覧できること。`http://192.168.0.100:30000`
- Redmineのデータがコンテナ**再起動時**にも保持されていること。

## 情報
- Server:
- k8smaster1:
    - ip: 192.168.0.100
    - userid: root
    - password: USerPw@19
- container-registry:
    - ip: 192.168.0.101
    - 備考: 操作不可
- Redmine_Manifest:
    - path: “/root/ictsc_problem_manifests/*.yaml”
- Redmineログイン情報
    - userid: ictsc
    - password: USerPw@19

## コメント
manifestファイルが欲しい〜〜〜〜〜〜

----

## 考えられる検証、修正手順

### バグの原因を特定する案
- `kubectl describe pods Redmine-Pod`
- `kubectl describe pods MariaDB-Pod` で何が原因で動かないかをはじめに調べる
- 問題文にこのような文章があったため、これに沿って設定を行ってみることにする

```
1. masterへpodのデプロイに関するtaintsの削除
2. コンテナランタイムcri-oにinsecure-registryの設定を追加
3. MariaDBのPersistentVolumeのディレクトリ権限(Permission)を修正
```

### masterへpodのデプロイに関するtaintsの削除
#### Taintsとは
- Tolerationsとセットで扱う

> taintは"汚れ"という意味。tolerationは"容認"という意味。つまり"汚れ"を"容認"できるならscheduleできる仕組み
> toleration はPodに適用され、一致するtaintが付与されたNodeへpodが不適当なnodeにscheduleされないようにする。
Node Affinityなどの場合には、それらが未指定の場合はどこのNodeにでもscheduleされてしまうが、TaintsとTolerationsの場合は指定しない限りそのNodeにscheduleされることはない。
1つもしくは複数のTaintsをnodeに設定することができ、podには1つもしくは複数のtolerationsを設定することができる。

```
kubectl taint nodes node1 key1=value1:NoSchedule
```
node1にはこんな感じで適用させる。

```
kubectl taint nodes node1 key1=value1:NoSchedule-
```
外すには以上のコマンドを利用する。

```
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
```
こんな感じでマニフェストファイルの書かれているため、削除自体はファイルを参考にできそう。

### コンテナランタイムcri-oにinsecure-registryの設定を追加
#### CRI-Oとは
CRI-Oとは、コンテナ型仮想化で使われる技術の1つで、Kubernetesとコンテナランタイムが通信するための仕様として規定されているCRI（Container Runtime Interface）と、OCI Runtime Specificationに基づいて作られたKubernetesやDockerの高レベルなランタイム。CNCFで開発が行われ、オープンソースソフトウェア。コード見たらGoで書かれてた。

#### insecure-registryの設定を追加
<b>insecure-registry</b> ... Registryとの非セキュアな通信を許可するオプションとして--insecure-registryオプションが存在する。Registryが暗号化されていないhttp通信の場合に必要な設定。

一つ目のドキュメントをみると、cri-oは`daemon.json`を使っているみたいなので、二つ目のドキュメントの参考をそのまま使うことができそう。

```
$ vi /etc/docker/daemon.json
{ "insecure-registries":["172.16.1.100:5000"] } # プライベートレジストリを指定 このリンクは任意
$ systemctl restart docker
$ systemctl restart crio # これもやっておいた方がいいかも？

# 起動確認例
$ docker pull 172.16.1.100:5000/ubuntu:16.04
$ docker run \
> -it \
> --rm \
> --name c1 \
> 172.16.1.100:5000/ubuntu:16.04 cat /etc/os-release
NAME="Ubuntu"
VERSION="16.04.2 LTS (Xenial Xerus)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 16.04.2 LTS"
VERSION_ID="16.04"

```
参考:
- https://kubernetes.io/ja/docs/setup/production-environment/container-runtimes/
- https://www.itmedia.co.jp/enterprise/articles/1708/25/news014_2.html

### MariaDBのPersistentVolumeのディレクトリ権限(Permission)を修正
これはよくわかんないけど、yamlを読んで修正をするのか、PersistentVolumeの`hostPath`の権限を見直すのかなのかな。よくわかんない。

```Sample.yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: my-pv-hostpath
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /data
```
-----
## 解説
流れまで書いてあって丁寧だなと思った。

## 解決手順
### masterへpodのデプロイに関するtaintsの削除
-  `kubectl get pod`でコンテナの状態を見ます。
```
[root@k8smaster1 ~]# kubectl get pod
NAME                                  READY   STATUS    RESTARTS   AGE
my-release-mariadb-0                  0/1     Pending   0          9d
my-release-redmine-859cf77958-n95j5   0/1     Pending   0          9d
```
→ Pendingになっていることがわかる

- `kubectl describe pod <pod名>`で各Podを確認する
    - あってた
```
Events:
  Type     Reason            Age                 From               Message
  ----     ------            ----                ----               -------
  Warning  FailedScheduling  9d (x5 over 9d)     default-scheduler  0/1 nodes are available: 1 node(s) had taints that the pod didn't tolerate.
```
- nodeのtaintsをpodが許容できないということなので、nodeのtaintsを`kubectl describe nodes`で確認します。
```
[root@k8smaster1 ~]# kubectl describe nodes
Name:               k8smaster1
Roles:              master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=k8smaster1
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/master=
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: /var/run/crio/crio.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Sat, 23 Nov 2019 19:58:55 +0900
Taints:             node-role.kubernetes.io/master:NoSchedule
```
- 一番最後の行で`node-role.kubernetes.io/master:NoSchedule`とあるため、Podがスケジューリングできない。
- Taintsを削除する
```
[root@k8smaster1 ~]# kubectl taint nodes k8smaster1 node-role.kubernetes.io/master:NoSchedule-
node/k8smaster1 untainted
```
⏫ こんな感じで`-`つけるだけでも削除できるんだね。便利〜〜〜。

### コンテナランタイムcri-oにinsecure-registryの設定を追加
- `kubectl get pod`で確認する
```
[root@k8smaster1 ~]# kubectl get pod
NAME                                  READY   STATUS             RESTARTS   AGE
my-release-mariadb-0                  0/1     ImagePullBackOff   0          9d
my-release-redmine-859cf77958-n95j5   0/1     ImagePullBackOff   0          9d
```
- また`describe`する
> Failed to pull image "private-registry.local/bitnami/mariadb:10.3.20-debian-9-r0": rpc error: code = Unknown desc = pinging docker registry returned: Get https://private-registry.local/v2/: dial tcp 192.168.0.101:443: connect: no route to host

- ホストへのルートがない？のかな


### MariaDBのPersistentVolumeのディレクトリ権限(Permission)を修正
- `kubectl get pod`をする
```
[root@k8smaster1 ~]# kubectl get pod
NAME                                  READY   STATUS    RESTARTS   AGE
my-release-mariadb-0                  0/1     Error     5          9d
my-release-redmine-859cf77958-n95j5   0/1     Running   1          9d
```
- mariadbのエラー
- ログを確認できるらしい
```
[root@k8smaster1 ~]# kubectl logs  my-release-mariadb-0
 16:43:21.98
 16:43:21.98 Welcome to the Bitnami mariadb container
 16:43:21.98 Subscribe to project updates by watching https://github.com/bitnami/bitnami-docker-mariadb
 16:43:21.98 Submit issues and feature requests at https://github.com/bitnami/bitnami-docker-mariadb/issues
 16:43:21.98 Send us your feedback at containers@bitnami.com
 16:43:21.99
 16:43:21.99 INFO  ==> ** Starting MariaDB setup **
 16:43:22.04 INFO  ==> Validating settings in MYSQL_*/MARIADB_* env vars
 16:43:22.04 INFO  ==> Initializing mariadb database
mkdir: cannot create directory '/bitnami/mariadb/data': Permission denied
```
- `create`ができない権限不足
    - これ見たらmkdirしようと思っちゃうけど解法は違う
- `/root/ictsc_problem_manifests`にあるk8sManifestを読み解くと、`/var/opt/pv{1,2}`にPersistentVolumeがある
    - そこに権限を渡せばいい。
- `kubectl get pv`の`mariadb`の対応するPathに権限を付与する
    - `kubectl get pv`で
```
[root@k8smaster1 ictsc_problem_manifests]# kubectl get pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                               STORAGECLASS   REASON   AGE
pv0001   20Gi       RWO            Recycle          Bound    default/data-my-release-mariadb-0                           9d
pv0002   20Gi       RWO            Recycle          Bound    default/my-release-redmine                                  9d
 
[root@k8smaster1 ]# chmod -R 777 /var/opt/pv1/
```

### 終わりに
`kubectl describe`は偉大。`Taints`理解した。
