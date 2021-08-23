# 適当に俳句投稿サービス作ったらXSRF脆弱性孕んでた件。

[適当に俳句投稿サービス作ったらXSRF脆弱性孕んでた件。](https://blog.icttoracon.net/2020/03/01/適当に俳句投稿サービス作ったらxsrf脆弱性孕んで/)

## 問題文

俳句投稿サービスHikerを作成した。Hikerでは、ユーザー作成後ログインして俳句を詠むことができる。詠んだ俳句は公開される。また、他のユーザーが詠んだ俳句に対してmogamigawaする機能（所謂お気に入り機能）がある。

ユーザーからのフィードバックで、意図しない俳句がmogamigawaされており困っているという情報が複数あった。それらのユーザーは共通して特定のWebサイトを閲覧したようである。
以上のことからHikerはXSRF脆弱性を孕んでいることが予想される。これらのユーザーはこの脆弱性を利用して、意図しない俳句をmogamigawaさせられたと考えられる。

任意の方法でこの脆弱性に対して対策を施してMerge Requestを建ててほしい。

## 使用環境・ツール

- サーバーサイド: Golang, Gin    // GinはGo言語の人気webフレームワーク
- フロントエンド: multitemplate + Bootstrap4
- データベース: MySQL + GORM
- docker-compose

## 初期状態

- 各チームのVMはHikeのステージング環境である。remoteRepositoryは削除されているので、各自でfork先urlを設定すること。
  - url: https://gitlab.com/ictsc2019-teamチーム番号/ictsc2019-f21-xsrf
  - urlのチーム番号部分には1,2,3,…,15のような自分のチーム番号を代入すること
- 既に`ictsc`というユーザーが俳句を投稿している
- Hikeで各自アカウントを新規作成後、ログインし https://hackmd.io/tRks_vT-QjasH9hUsEJ-BA?view にアクセスすると`ictsc`というユーザーの俳句をmogamigawaしてしまう
- ソースコードはGitLabで管理されており、問題解答開始時にチームリーダーにOWNER権限のinviteのメールが送信される
- Hikerが動いているときは、http://192.168.15.1:8080 でサービスにアクセスできる

## 終了状態

- 適切なXSRF対策がされている
- 初期状態に示されているURL上の検証コードはあくまで一例であることに注意すること
- 修正されたソースコードのMerge RequestをGitLab上で作成する   // Merge RequestはGitHubでいうPull Request
- スコアサーバに、Merge RequestのURLを提出する

----

## 解決方法

### CSRFとは

CSRF ... Cross-Site Request Forgeries／クロスサイト・リクエスト・フォージェリ（偽サイトを使ってリクエストを偽造する）

https://medium-company.com/%E3%82%AF%E3%83%AD%E3%82%B9%E3%82%B5%E3%82%A4%E3%83%88%E3%83%AA%E3%82%AF%E3%82%A8%E3%82%B9%E3%83%88%E3%83%95%E3%82%A9%E3%83%BC%E3%82%B8%E3%82%A7%E3%83%AA/

他の参考サイト：

https://www.ipa.go.jp/security/vuln/websecurity-HTML-1_6.html

https://qiita.com/wanko5296/items/142b5b82485b0196a2da#csrf%E3%81%A8%E3%81%AF%E4%BD%95%E3%81%8B

### 解決の流れ

>CSRFへの代表的な対策
>
>**Formページ返却時のトークン付与**
>今回の例でいうと、はじめに掲示板への書き込み画面を表示する際にサーバがクライアントに対して特定の文字列（トークン）を設定します。実際に書き込みのリクエストがあった際にサーバーが**「この人に送ったトークンと同じトークンがリクエストに入ってる？」**と確認することで、攻撃者からの不正なリクエストを防ぐことができます。これは、攻撃者は利用者に送信したトークンの値を知らないためです。

1. mogamigawaする画面を要求されたら、暗号論的擬似乱数生成器を用いて機密情報を作るようにする
2. 機密情報も入れてmogamigawaする画面を返すようにする
3. mogamigawaするときに、hiddenタグで機密情報も送るようにする
4. セクションIDと機密情報があっているか確認するようにする

## 解説

ハッカソン的なイベントでよく適当にwebサービスを作ると思います。作りますね。そんなときの*あるある*ですが、割とWebのセキュリティを考えずにデプロイして成果発表みたいなノリです。良くないですね。そんな問題でした。

XSRF脆弱性の対策をします。様々な手法が考えられますが、今回はWeb Application Framework(WAF)にGinを採用しているので、utrack/gin-csrfを使って対策するケースで解説します。
utrack/gin-csrfで実装する理由は、問題環境はステージング環境でありdocker-composeで管理されていることから、様々な場所で実行されることを考慮して、utrack/gin-csrfのような環境に依存しにくい実装をしたいためです。

### リモートリポジトリを追加する

ステージング環境にデプロイされているソースコードのディレクトリに移動してリモートリポジトリを追加します。

```
$ git remote add origin https://gitlab.com/ictsc2019-teamチーム番号/ictsc2019-f21-xsrf
```

この問題では解答にMerge Requestを建てなければいけないので、cloneしたときかこのタイミングでbranchを切ります。

```
$ git checkout -b xsrf-fix
$ git push --set-upstream origin xsrf-fix   // これを書くと毎回origin xsrf-fixの部分を書かなくて良くなるっぽい
```

あとはcommit~~と徳~~を積んでMerge Requestをします。

### `server.go` の修正

utrack/gin-csrfの`README.md`を参考に頑張ります。 // https://github.com/utrack/gin-csrf#readme
変更点は次の通りです。

#### import部分の追記

- `"net/http"`を追加しました。csrfのErrorFuncで`http.StatusBadRequest`を使用するためです。
- `"github.com/utrack/gin-csrf"`を追加しました。gin-csrfを使います。
- `"ictsc2019-f21-xsrf/util"`を追加しました。csrfのSecretをランダムな文字列にする関数を`app/util/util.go`に追記して、それを使用するためです。

```
import (
    "log"
    "net/http"
    "path/filepath"
 
    "github.com/gin-contrib/multitemplate"
    "github.com/gin-contrib/sessions"
    "github.com/gin-contrib/sessions/cookie"
    "github.com/gin-gonic/gin"
    csrf "github.com/utrack/gin-csrf"
 
    "ictsc2019-f21-xsrf/domain"
    "ictsc2019-f21-xsrf/handler"
    "ictsc2019-f21-xsrf/util"
)
```

#### func main部分の変更

以下の記述を追加します。utrack/gin-csrfの`README.md`の通りです。

```
// csrf
r.Use(csrf.Middleware(csrf.Options{
    Secret: util.RandString(10),
    ErrorFunc: func(c *gin.Context) {
        c.JSON(http.StatusBadRequest, gin.H{
            "error": "CSRF token mismatch",
        })
        c.Abort()
    },
}))
```

そして、mogamigawaするAPIをPOSTに変更します。

```
apiRouter.POST("/mogamigawa", handler.NewMogamigawa)
```

以上が`server.go`の更新作業になります。

### `util.go` に追記

`server.go`で呼び出されている`util.RandString(n int)`を、`util.go`に作成します。
`"math/rand"`を追加でimportしてください。

```
const rs2Letters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
 
func RandString(n int) string {
    b := make([]byte, n)
    for i := range b {
        b[i] = rs2Letters[rand.Intn(len(rs2Letters))]
    }
    return string(b)
}
```

RandString(n int)を叩くことでn文字のランダムな文字列を返すことができます。これをcsrfのSecretに使います。

このSecretですが、文字列が固定されている解答がありました。これは第三者から推測が困難ではないかもしれないので適切ではありません。(減点はしてません)

RandString(n int)を叩くことでn文字のランダムな文字列を返すことができます。これをcsrfのSecretに使います。

このSecretですが、文字列が固定されている解答がありました。これは第三者から推測が困難ではないかもしれないので適切ではありません。(減点はしてません)

### XSRF対策をする

今回は全てのFORM要素にhiddenなinputを用意して、そこにtokenを持たせ、送信させることにします。

#### `hikeline.html` の修正

17行目にhiddenなinputを追加します。

```
<form method="POST" action="api/newhike">
<input type="hidden" name="_csrf" value="{{ .csrfToken }}">
```

また、`hikeline.html`のmogamigawaのbutton部分は以下のように入れ替えます。この変更でmogamigawaのAPIをPOSTにした変更に対応し、XSRF対策ができます。

```
<!-- mogamigawa button -->
<div>
  <form action="/api/mogamigawa?hike_id={{ .Hike_id }}" method="POST">
    <input type="hidden" name="_csrf" value="{{ $.csrfToken }}">
    <button type="submit">
      <a href=""><i class="fas fa-water"></i></a>
    </button>
    <style>
      button {
        padding: 0, 0;
        border-style: none;
      }
 
    </style>
  </form>
</div>
```

`signin.html`と`signup.html`にもformがありますが、ここはXSRFから保護しなければならない場所ではありませんよね?

55%の採点を受けたチームは、この部分で`api/mogamigawa`の対策はできてるんだけど、`api/newhike`の対策がなされてない解答になっていました。

以上がhtmlの修正作業になります。

#### `res.go`の修正

htmlに`{{ .csrfToken }}`という新しいプレースホルダーを追加しました。これに対応して`/app/handler/res.go`を更新します。

`"github.com/utrack/gin-csrf"`を追加でimportしてください。gin-csrfを使います。
`hikeline.html`を返す部分の`c.HTML(http.StatusOK, hikeline.html, gin.H{})`に、以下のように記述を追加します。冗長なので、全箇所の記述は省略します。

```
c.HTML(http.StatusOK, hikeline.html, gin.H{
"csrfToken": csrf.GetToken(c),
})
```

### Merge Request

以上の変更をcommitしたらpushして、GitLabでMerge Requestを建てます。

解説は以上です。

## 採点基準

1. 適切なMerge Requestがなされている: +10%
   - あまりにも杜撰な解答は許されません。:angry:
2. 任意の手法でXSRF対策をしている: +90%
   - mogamigawaのAPIとフロントエンドにのみ修正を加えた場合は半分の +45% にしています。
   - mogamigawaのAPIに加えてXSRF対策の必要なnewhikeのAPIにも対策をした場合に +90% としました。
   - 適切なXSRF対策がなされている場合にのみ点数を取れるようにしました。

## 講評

10チームが解答を提出してくれました。