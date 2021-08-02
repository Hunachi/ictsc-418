# APIが飛ばないんですけど

[APIが飛ばないんですけど…](https://blog.icttoracon.net/2019/09/01/ictsc2019-一次予選-問題解説-apiが飛ばないんですけど/)

## 問題文

Webアプリケーションからドメインが異なるAPIにリクエストを発行する際には、クロスオリジンについて注意する必要があります。
CORS (Cross-Origin Resource Sharing) に関する以下の問いについて、それぞれ適切な選択肢を選んでください。

## 問１

### 問題文

`https://example.com` と同じOriginを選んで下さい。

- A. https://example.com/hoge
- B. [https://example.com:8080](https://example.com:8080/)
- C. [https://hoge.example.com](https://hoge.example.com/)
- D. [http://example.com](http://example.com/)

### 解答

> **オリジン Origin** は、ウェブコンテンツにアクセスするために使われる URL のスキーム (プロトコル)、 ホスト (ドメイン)、 ポート によって定義されます。 スキーム、ホスト、ポートがすべて一致した場合のみ、二つのオブジェクトは同じ**オリジン**であると言えます。

A

理由：

B サーバーは既定で80番ポートで HTTP コンテンツを配信するため `https://example.com` は `https://example.com:80` と同じ

C ホストが異なる

D プロトコルが異なる

### 解説

ポート番号、プロトコル（HTTP か HTTPS か）、ホストが一致するときのみ同一のOriginとなります。したがって `https://example.com/hoge` のみが正解です。

参考: https://developer.mozilla.org/ja/docs/Glossary/Origin



## 問２

### 問題文

`app.ictsc` で動いているアプリケーションから `api.ictsc` へ以下のような `fetch()` を実行したところ、CORSのエラーで正常に動きませんでした。 `api.ictsc` に設定する必要があるHTTP response headerをすべて選んでください。

```
fetch({
  method: "POST",
  headers: {
    "Content-Type": "application/json"
  },
  "body": JSON.stringify(data)
})
```

- Access-Control-Allow-Origin
- Access-Control-Allow-Headers
- Access-Control-Allow-Methods

### 解答

全部

CORS (Cross-Origin Resource Sharing) とは...：

> あるオリジンで動いている Web アプリケーションに対して、別のオリジンのサーバーへのアクセスをオリジン間 HTTP リクエストによって許可できる仕組みのこと

Access-Control-Allow-Origin：

> 指定された[オリジン](https://developer.mozilla.org/ja/docs/Glossary/Origin)からのリクエストを行うコードでレスポンスが共有できるかどうかを示します。

Access-Control-Allow-Headers：

> 実際のリクエストの間に使用できる HTTP ヘッダーを示すために使用されます。

Access-Control-Allow-Methods

> リソースにアクセスするときに利用できる1つまたは複数のメソッドを指定します。
>
> 例：
>
> ```
> Access-Control-Allow-Methods: POST, GET, OPTIONS
> Access-Control-Allow-Methods: *
> ```

### 解説

問題文中の fetch では `https://api.ictsc` へ `https://app.ictsc` から POST リクエストが実行されます。これはホストが異なるため、異なるOriginへのリクエストになるので、応答の HTTP ヘッダに `Access-Control-Allow-Origin` が必要です。

リクエストには `Content-Type` ヘッダが含まれており、その値が `application/json` になっています。`Content-Type` が以下の3つの値以外のときは、実際のリクエストの前に preflight request が発行されます。

- `application/x-www-form-urlencoded`
- `multipart/form-data`
- `text/plain`

preflight request は `OPTIONS`メソッドで行われ、サーバ側の許可するメソッドやヘッダが応答の HTTP ヘッダ内の情報として返されます。問題文のように `Content-Type: application/json` のリクエストを送る場合は、サーバ側で prefilght request への応答の HTTP ヘッダに `Access-Control-Allow-Headers: Content-Type` と設定しておく必要があります。

一方で、メソッドが`GET`, `HEAD`, `POST` の場合は preflight request への応答の HTTP ヘッダに `Access-Control-Allow-Methods` をつける必要はありません。

したがって、設定するべきヘッダは `Access-Control-Allow-Origin` と `Access-Control-Allow-Headers` となります。

参考: https://developer.mozilla.org/ja/docs/Web/HTTP/CORS#Preflighted_requests

## 問３

### 問題文

選択肢に示すHTTPメソッドのうち、いかなる場合においてもpreflight requestが行われるものを選んでください。

- GET
- POST
- HEAD
- DELETE

### 解答

DELETE

CORS のプリフライトリクエストは [CORS](https://developer.mozilla.org/ja/docs/Glossary/CORS) のリクエストの一つであり、サーバーが CORS プロトコルを理解していて準備がされていることを、特定のメソッドとヘッダーを使用してチェックします。

プリフライトリクエストはブラウザーが自動的に発行するものであり、通常は、フロントエンドの開発者が自分でそのようなリクエストを作成する必要はありません。これはリクエストが["to be preflighted"](https://developer.mozilla.org/ja/docs/Web/HTTP/CORS#preflighted_requests)と修飾されている場合に現れ、[単純リクエスト](https://developer.mozilla.org/ja/docs/Web/HTTP/CORS#simple_requests)の場合は省略されます。

「単純リクエスト」は、**以下のすべての条件を満たす**ものです。

- 許可されているメソッドのうちの一つであること。
  - [`GET`](https://developer.mozilla.org/ja/docs/Web/HTTP/Methods/GET)
  - [`HEAD`](https://developer.mozilla.org/ja/docs/Web/HTTP/Methods/HEAD)
  - [`POST`](https://developer.mozilla.org/ja/docs/Web/HTTP/Methods/POST)
- ...

### 解説

preflight request の概要については問2の解説で説明したとおりです。メソッドが以下に挙げるものの場合は、必ず preflight request が発行されます。

- `PUT`
- `DELETE`
- `CONNECT`
- `OPTIONS`
- `TRACE`
- `PATCH`

したがって正解は `DELETE` となります。

## 問４

### 問題文

preflight requestについて示した文章のうち、正しいものを全て選んでください。

- リクエスト元のドメインとリクエスト先のドメインが同じ場合は、いかなる場合においてもpreflight requestは行われない。
- クロスオリジンで独自HTTPメソッド `TEST` を発行するためには、`Access-Control-Allow-Methods` に `*` を追加することで必ず正しく動く。
- preflight requestに対する応答は、`Access-Control-Allow-*` ヘッダの内容が正しいHTTP responseであれば他の内容はなんでもよい。
- `Access-Control-Allow-Origin` に `*` を設定しておけば、他のヘッダが適切である限りいかなる場合でも動作する。

### 解答

x x x o

- Access-Control-Allow-Methods
  - `*` (ワイルドカード)
  - "`*`" の値は、資格情報のないリクエスト ([HTTP Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies) や HTTP 認証情報のないリクエスト) の特殊なワイルドカードです。

### 解説

> リクエスト元のドメインとリクエスト先のドメインが同じ場合は、いかなる場合においてもpreflight requestは行われない。

ドメインが同じであってもOriginが同じであるとは限りません。ポート番号やプロトコルが異なる場合は異なるOriginとなります。Originが異なる場合、特定の条件を満たせばpreflight requestが行われるため、この文章は間違っています。



> クロスオリジンで独自HTTPメソッド `TEST` を発行するためには、`Access-Control-Allow-Methods` に `*` を追加することで必ず正しく動く。

`Access-Control-Allow-Methods: *` と設定した場合の独自メソッドの動作は実装に依存しています。Ubuntu 18.0.4 上で Python3.6.8 の Bottle v0.12.17 によりHTTPサーバを`http://localhost:8090`, `http://localhost:8080`に立てて、前者から後者に JavaScript の fetch でリクエストを送って検証しました。Chromium 76.0.3809 で `TEST`リクエストを行ってみると成功しますが、FireFox 68.0.1 では失敗しました。 したがって、「必ず正しく動く」とするこの文章は間違っています。

参考: https://developer.mozilla.org/ja/docs/Web/HTTP/CORS/Errors/CORSMethodNotFound



> preflight requestに対する応答は、`Access-Control-Allow-*` ヘッダの内容が正しいHTTP responseであれば他の内容はなんでもよい。

preflight request に対する応答のステータスコードが200番台でない場合、リクエストを送ることができません。上記と同様の検証環境で、preflight requestに対するHTTP responseのステータスコードを404にするとPOSTリクエストが飛ばないことを確かめられました。したがってこの文章は間違っています。

参考:[ ](https://www.w3.org/TR/cors/#preflight-request)https://fetch.spec.whatwg.org/#cors-preflight-fetch



> `Access-Control-Allow-Origin` に `*` を設定しておけば、他のヘッダが適切である限りいかなる場合でも動作する。

リクエストにCookieなどのリクエスト情報が含まれている場合、`Access-Control-Allow-Origin: *`というワイルドカードの指定ではなく、具体的なOriginの指定が必要です。したがって、「いかなる場合でも動作する」とするこの文章は間違っています。

参考: https://developer.mozilla.org/ja/docs/Web/HTTP/CORS#Requests_with_credentials

以上より、全ての文が間違っているので、何も選択しないのが正解です。