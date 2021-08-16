# すごく匿名ダイヤリー

[すごく匿名ダイヤリー](https://blog.icttoracon.net/2019/12/10/ictsc2019-二次予選-問題解説-すごく匿名ダイヤリー/)

## 問題文

匿名で日記が投稿できるサービス「すごく匿名ダイヤリー」を運営しています。
従来、フロントエンドとバックエンドを同じドメインで運用していましたが、
構成変更のため、バックエンドをサブドメインに変更する作業を行っています。

変更前:
https://old-diary.ictsc.net/
https://old-diary.ictsc.net/api/

変更後:
https://new-diary.ictsc.net/
https://api.new-diary.ictsc.net/

※VNCサーバのWebブラウザからのみ閲覧可能です

ソースコード内のドメインやパスは適切に書き換えましたが、何故か正常に動作しません。
変更前と同じように各機能が動作するよう、サーバにログインして原因調査 及び 修正を行ってください。

なお、サービスはメンテナンス中で限定公開としているため、対応中にサービス断が生じても問題ありません。
また、投稿データについてもバックアップから復元するので、(変更前/変更後環境共に)日記の追加・削除・スター追加は任意に実施して問題ありません。

今後の運用・開発を考慮し、変更は問題解決に必要な箇所に絞り、出来るだけ他に影響を与えないように直してください。
全てを直しきれない場合でも、可能なところまで直してください。

### サービス仕様

- 誰でも匿名で日記が投稿・閲覧できる
- 投稿されている日記に対して誰でもスターを付けることができる
- 日記は投稿したブラウザで閲覧すると削除ボタンが表示され、削除が可能 (期間/個数に制限あり)
- フロントエンドはSPA(Single Page Application)として構築されている
- 日記の取得/投稿/削除/スター追加はWebAPI経由でバックエンドと通信して実現する

// SPAとは（https://qiita.com/takanorip/items/82f0c70ebc81e9246c7a）：

> JavaScriptでDOMを操作しページを切り替える

### 解答方法

- 修正 と 報告 の両方が必要です
- 「変更後」のURLでサービスが正常に動作するよう、実際にサーバ上で修正を行ってください。
- 解答から「原因と実施した修正内容」を報告してください。
  - 報告は最終的に行った内容のみで問題ありません (途中の試行錯誤は記載不要)
  - 具体的に記載してください (例: XXXを直した、ではなく XXXがXXXなので、XXXファイルのXXX部分にXXXXXXXXXを追加した 等)

### ログイン情報

VNCサーバから
$ ssh 192.168.0.80 -l admin
→ PW: USerPw@19

※ $ sudo su – にて rootユーザに昇格可能です

----

## 考えられる検証、修正手順

> 何故か正常に動作しません。

どこが正常に動いてないのか調べて、エラーメッセージを読めたら読む。

> ソースコード内のドメインやパスは適切に書き換えましたが

DNS周りは大丈夫なのか調べる。

----

## 解説

この問題はICTSC2019 一次予選にて出題された [APIが飛ばないんですけど…](https://blog.icttoracon.net/2019/09/01/ictsc2019-一次予選-問題解説-apiが飛ばないんですけど/) の実技出題を目的として作成しました。

// 完全に気づかなかった ／(^O^)＼

機能ごとに必要な対処が異なり、CrossOrigin通信におけるCORS, CSP, Cookieの取り扱いを把握していないと完答出来ない構成としています。

// CSPとは：セキュリティの為のHTTPレスポンスヘッダー (https://techblog.securesky-tech.com/entry/2020/05/21/)

### STEP1, 日記一覧と日記を閲覧可能にする 前半 (CSPによる許可)

// https://new-diary.ictsc.net/しか信用できませんという設定になっているため、https://api.new-diary.ictsc.netを信用するようにする

https://new-diary.ictsc.net/ を閲覧するとブラウザアラートで`Error: Network Error`と表示されます。
これだけでは原因がわからないので、開発者ツール(F12)のコンソールを表示すると以下のエラーが表示されています。

```
Content Security Policy: ページの設定により次のリソースの読み込みをブロックしました: https://api.new-diary.ictsc.net/list (“connect-src”)
```

→ CSPの “connect-src” で https://api.new-diary.ictsc.net/list への接続が禁止されていることが分かります。

ページのソースを表示するとmetaタグでCSPが指定されている為、このhtmlを修正する必要があります。

```
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; connect-src 'self'; script-src 'self' 'unsafe-eval'; style-src 'self' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com;">
```

// connect-src <ドメイン>で、<ドメイン>のスクリプトだけを信用するという意味になる

修正すべきファイルの場所は動作しているWebサーバの設定ファイルから特定します。

```
# netstat -ntelpo | grep -e :443
// 接続待ち(Listen)のプロセスを調べる(?) | ポート番号は、httpsのデフォルトのポート番号である443に絞る
tcp6       0      0 :::443                  :::*                    LISTEN      0          26477      2044/httpd           off (0.00/0/0)
// httpdとは：dはデーモン　Webサーバとしてのお仕事をしている常駐プログラム
# ps auxww | grep http[d]
// 全ユーザの全プロセス | httpdに絞る []で自分自身を避けてるらしい(https://webkaru.net/linux/ps-grep-exclude/)
root      2044  0.0  1.3 286180 13864 ?        Ss   17:59   0:00 /usr/sbin/httpd -DFOREGROUND
apache    2778  0.0  0.9 298736  9104 ?        S    18:48   0:00 /usr/sbin/httpd -DFOREGROUND
apache    2779  0.0  1.5 1356304 15728 ?       Sl   18:48   0:00 /usr/sbin/httpd -DFOREGROUND
apache    2780  0.0  1.6 1356172 16760 ?       Sl   18:48   0:00 /usr/sbin/httpd -DFOREGROUND
apache    2781  0.0  1.7 1487424 18028 ?       Sl   18:48   0:00 /usr/sbin/httpd -DFOREGROUND
apache    2993  0.0  1.6 1356308 17012 ?       Sl   18:48   0:00 /usr/sbin/httpd -DFOREGROUND
# /usr/sbin/httpd -S 2>&amp;1 | grep port
         port 443 namevhost fe80::9ea3:baff:fe30:1584 (/etc/httpd/conf.d/ssl.conf:40)
         port 443 namevhost old-diary.ictsc.net (/etc/httpd/conf.d/virtualhost.conf:6)
         port 443 namevhost new-diary.ictsc.net (/etc/httpd/conf.d/virtualhost.conf:32)
         port 443 namevhost api.new-diary.ictsc.net (/etc/httpd/conf.d/virtualhost.conf:51)
// Webサーバの設定ファイルは/etc/httpd/conf.d/virtualhost.conf
# grep Root -B1 /etc/httpd/conf.d/virtualhost.conf
  ServerName old-diary.ictsc.net
  DocumentRoot /var/www/old-front
--
  ServerName new-diary.ictsc.net
  DocumentRoot /var/www/new-front
--
  ServerName api.new-diary.ictsc.net
  DocumentRoot /var/www/new-api/public
```

`/var/www/new-front/index.html` に該当のmetaヘッダが含まれている為、
`connect-src 'self';`を`connect-src https://api.new-diary.ictsc.net;` に編集すると、問題のエラーが解消します。

### STEP2, 日記一覧と日記を閲覧可能にする 後半 (CORSによる許可)

// https://new-diary.ictsc.net オリジンを許可する

STEP1でCSPによるエラーは解消しましたが、まだ閲覧可能にはなりません。
再び https://new-diary.ictsc.net/ を開いてコンソールを確認すると、以下のエラーが表示されます。

```
クロスオリジン要求をブロックしました: 同一生成元ポリシーにより、https://api.new-diary.ictsc.net/list にあるリモートリソースの読み込みは拒否されます (理由: CORS ヘッダー ‘Access-Control-Allow-Origin’ が足りない)。
```

記載の通り、CORSヘッダーの設定が必要となります。
https://developer.mozilla.org/ja/docs/Web/HTTP/CORS
設定場所についてはいくつか考えられますが、作問者の想定は以下の2通りです。

#### アプリケーション側に追加

`/var/www/new-api/public/index.php` の `FastRoute\Dispatcher::FOUND` 以下等に追加する

```
case FastRoute\Dispatcher::FOUND:
    $handler = $routeInfo[1];
    $vars = $routeInfo[2];
    header('Access-Control-Allow-Origin: https://new-diary.ictsc.net'); ★ 追加
```

#### Webサーバ(Apache)側に追加

`/etc/httpd/conf.d/virtualhost.conf` の `<Directory /var/www/new-api/public>` 内等に追加し、httpdをreloadする

```
<Directory /var/www/new-api/public>
  Options Indexes FollowSymLinks
  AllowOverride All
  Require all granted
 
  Header set Access-Control-Allow-Origin https://new-diary.ictsc.net ★ 追加
</Directory>
```

※ 本問題ではブラウザ上で各機能が正しく動作していれば、追加場所や細かい記載方法等は不問としました。
※ ただし、アプリケーションを1から作り直すような大幅な変更は認めていません。

以上の変更を行うと、日記一覧 及び 日記が閲覧可能となります。

### STEP3, 日記の投稿を可能にする

// ヘッダを追加

「日記を書く」から日記を投稿すると、ブラウザアラートで`投稿後の日記URLが受け取れませんでした。`と表示されます。
また、コンソールには`submit_article https://new-diary.ictsc.net/app.js:109` と表示されます。
ただし、日記の投稿は正常に完了しており、その後のページ遷移のみ失敗しているようです。

エラーメッセージだけでは情報が足りないので、`https://new-diary.ictsc.net/app.js`の該当処理を確認すると、
`res.headers.location`、つまりレスポンスのLocationヘッダが正常に取得出来ていないようです。

```
axios.post(api_url + 'article', params)
    .then(res => {
    if (!res.headers.location) { throw `投稿後の日記URLが受け取れませんでした。` }
    router.push(res.headers.location)
    })
    .catch(err => { console.error(err); alert(err) })
}
```

一方、開発者ツールのネットワークタブでAPIサーバからの応答を確認すると、
日記投稿後、`Location: /article/21` のようにLocationヘッダを含むレスポンスが得られていると確認出来ます。

この解決には知識が必要となりますが、CORSでセーフリスト以外のレスポンスヘッダを利用する場合、
`Access-Control-Expose-Headers` ヘッダにて明示的に許可する必要があります。
https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Access-Control-Expose-Headers
Locationヘッダはセーフリストに含まれていない為、STEP2の設定に以下のヘッダも追加する必要があります。

```
Access-Control-Expose-Headers: Location
```

ヘッダを追加すると、日記投稿後のエラーが解消し、投稿された日記ページにリダイレクトされるようになります。

### STEP4, スターの追加を可能にする

// 定義されていないメソッドに対応

各記事のスター追加ボタン[★+]をクリックすると`Error: Network Error`が表示されます。
開発者ツールのコンソールには以下のように表示されます。

```
クロスオリジン要求をブロックしました: 同一生成元ポリシーにより、https://api.new-diary.ictsc.net/article/26/star にあるリモートリソースの読み込みは拒否されます (理由: CORS ヘッダー ‘Access-Control-Allow-Origin’ が足りない)。
クロスオリジン要求をブロックしました: 同一生成元ポリシーにより、https://api.new-diary.ictsc.net/article/26/star にあるリモートリソースの読み込みは拒否されます (理由: CORS 要求が成功しなかった)。
```

また、開発者ツールのネットワークタブで通信を確認すると、
`OPTIONS`メソッドのリクエストが送信され、`HTTP/1.1 405 Method Not Allowed`のレスポンスが得られています。
しかし、`https://new-diary.ictsc.net/app.js`にて利用されているメソッドは`PUT`です。

```
add_star: function () {
    axios.put(api_url + 'article/' + this.$route.params.id + '/star')
        .then(res => {
        this.article.star_count++
        })
        .catch(err => { console.error(err); alert(err) })
},
```

これは一次予選でも出題された プリフライトリクエストによる挙動です。
https://developer.mozilla.org/ja/docs/Web/HTTP/CORS#Preflighted_requests

`OPTIONS`メソッドに対して適切なCORSヘッダを応答する必要がありますが、
`/var/www/new-api/public/index.php`内で`OPTIONS`メソッドが定義されていない為、`METHOD_NOT_ALLOWED`として405の応答が発生しています。

作問者の想定解法は以下の2通りです。

#### ダミールートの追加

`/var/www/new-api/public/index.php` に ダミーのルートを追加する

```
$base = '/';
$dispatcher= FastRoute\simpleDispatcher(function(FastRoute\RouteCollector $router) use ($base) {
    $router->addRoute('GET'    , $base.'list'                  , 'get_list');
    $router->addRoute('GET'    , $base.'article/{id:\d+}'      , 'get_article');
    $router->addRoute('POST'   , $base.'article'               , 'post_article');
    $router->addRoute('DELETE' , $base.'article/{id:\d+}'      , 'delete_article');
    $router->addRoute('PUT'    , $base.'article/{id:\d+}/star' , 'put_article_star');
    $router->addRoute('OPTIONS', $base.'{path:.*}'             , 'dummy');   ★ 追加
});
 
function dummy($vars, $pdo) {  ★ 追加
    return;
}
```

合わせてCORSヘッダの設定箇所に以下を追加する必要があります。

```
Access-Control-Allow-Methods: GET, POST, PUT, OPTIONS
```

#### METHOD_NOT_ALLOWED発生時の処理に追加

`/var/www/new-api/public/index.php`で`METHOD_NOT_ALLOWED`が発生時した場合も、OPTIONSメソッドについては応答するように追加する

```
case FastRoute\Dispatcher::METHOD_NOT_ALLOWED:
    $allowedMethods = $routeInfo[1];
    if ($httpMethod == 'OPTIONS') {  ★ 追加
        header('Access-Control-Allow-Methods: OPTIONS, '.implode(', ', $allowedMethods));
        header('Access-Control-Allow-Origin: https://new-diary.ictsc.net');
        header('Access-Control-Expose-Headers: Location');
        break;
    }
    header('Allow: '.implode(', ', $allowedMethods));
    header('HTTP/1.1 405 Method Not Allowed');
    break;
```

上記どちらかの修正を行うと、スターの追加が可能となります。

### STEP5, 日記の削除を可能にする

// Cookieに正しいsecret(パスワード)が保存されるように設定する

ここまでの対処でブラウザ操作で発生するエラーは解消しました。
しかし、問題文に書かれている日記の削除機能が見当たりません。

- 日記は投稿したブラウザで閲覧すると削除ボタンが表示され、削除が可能 (期間/個数に制限あり)

`https://new-diary.ictsc.net/app.js` を確認すると、UI自体は存在するようですが、
`article.authored`が`true`にならなければ表示されないようです。

```
<div><span v-if="article.authored" class="delete_btn" v-on:click="delete_article()">この日記を削除する</span></div>
```

`https://new-diary.ictsc.net/app.js`には`article.authored`を変更する処理が含まれておらず、
APIからの結果をそのまま受け入れています。

```
mounted: function () {
    axios.get(api_url + 'article/' + this.$route.params.id)
    .then(res => {
        if (!res.data) { throw `日記が見つかりませんでした。` }
        this.article = res.data;
    })
    .catch(err => { console.error(err); alert(err) })
},
```

API側の処理を `/var/www/new-api/public/index.php` から確認すると、
Cookieに正しいsecret(パスワード)が保存されている場合のみ、`article.authored`が`true`となることが分かります。

```
function get_article($vars, $pdo) {
    $articleid = $vars['id'];
 
    $stmt = $pdo->prepare('SELECT id, title, content, star_count, secret_hash FROM article WHERE id = :id');
    $stmt->execute(array(':id' => $articleid));
    $result = $stmt->fetch();
    if (isset($_COOKIE['__Secure-article-'.$articleid])) {
        $secret_hash = $result['secret_hash'];
        $client_secret = $_COOKIE['__Secure-article-'.$articleid];
        $authored = password_verify($client_secret, $secret_hash);
    } else {
        $authored = false;
    }
```

記事の投稿時には`setcookie`が行われており、レスポンスヘッダからも確認できますが、
実際に投稿してもブラウザのCookieには保存されません。※ 開発者ツールのストレージタブにて確認出来ます。

```
function post_article($vars, $pdo) {
...
    header('HTTP/1.1 201 Created');
    header('Location: /article/'.$articleid);
    setcookie('__Secure-article-'.$articleid, $secret, time() + (365 * 86400), '
```

CrossOriginでCookieを設定させる場合は、リクエスト側で`withCredentials`の指定と、
レスポンス側で`Access-Control-Allow-Credentials`の指定が必要となります。
https://developer.mozilla.org/ja/docs/Web/API/XMLHttpRequest/withCredentials
https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Access-Control-Allow-Credentials

レスポンス側はこれまでのCORSヘッダと同様に以下のヘッダを追加します。

```
Access-Control-Allow-Credentials: true
```

リクエスト側については、`/var/www/new-front/app.js` からaxiosを利用して通信している為、
個別に`withCredentials: true`を指定するか、以下のようにデフォルト値を設定します。

```
axios.defaults.withCredentials = true;
```

双方を追加後に記事を投稿すると「この日記を削除する」ボタンが表示されるようになります。
実際の削除についてはDELETEメソッドを許可する必要があるため、追加していない場合はヘッダに追加します。

```
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
```

以上で全ての機能が正常に動作するようになりました。
動作確認の上、「原因と修正内容」を解答すれば完了です。

## 採点結果について

本問題は「各機能の正常な動作」及び「修正箇所への言及」にて点数を加算しています。

各工程の正答率は「STEP1/2 41%」「STEP3 21%」「STEP4 23%」「STEP5 17%」となり、完答は「12%」でした。
STEP1/2までの修正についてはWebブラウザの開発者ツール(コンソール)で修正箇所が示されていますので、
普段から使い慣れている方は比較的容易に解決できる想定でした。
一方、STEP3/4/5についてはCORS/Cookieの知識 及び PHP/JavaScriptの読解が必要となる為、
Web技術に関するチームの実力差が顕著に出る結果となったように感じます。
特に上位チームは解答内容が丁寧かつ明確な内容で、完全に理解している様子でした。
（拙いコードを読解いただきありがとうございました……）

なお、全ての問題に対処出来たと思われるチームでも、
「解答で一部修正に言及していない」「デバッグ用のalertが削除されないまま残っている」
「解答では修正されているはずのファイルがサーバ上では修正されていない」等の理由で減点が発生しました。
また、STEP1/2の解決のみで問題クリアと判断した様子のチームも見受けられました。

いずれも解答提出前後の見直しで防げる内容となりますので、
今一度落ち着いて問題文と解答、修正後のサービス状況を確認いただければと思います。