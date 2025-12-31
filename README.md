# Node.jsでUser Agentを設定・変更する方法

[![Promo](https://github.com/bright-jp/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.jp/) 

このガイドでは、Node.jsで`User-Agent`ヘッダーを設定する方法と、アンチボット検知を回避するためにユーザーエージェントローテーションを実装する方法を、[scraping with node.js](https://brightdata.jp/blog/how-tos/web-scraping-with-node-js)を行う際の観点から解説します。

- [Fetch APIを使用してNode.jsのUser Agentを変更する方法](#how-to-change-the-nodejs-user-agent-using-the-fetch-api)
  - [ローカルにUser Agentを設定する](#set-a-user-agent-locally)
  - [グローバルにUser Agentを設定する](#set-a-user-agent-globally)
- [Node.jsでUser Agentローテーションを実装する](#implement-user-agent-rotation-in-nodejs)
  - [ステップ1: User Agentのリストを取得する](#step-1-retrieve-a-list-of-user-agents)
  - [ステップ2: User Agentをランダムに選択する](#step-2-randomly-pick-a-user-agent)
  - [ステップ3: ランダムなUser AgentでHTTPリクエストを行う](#step-3-make-the-http-request-with-a-random-user-agent)
  - [ステップ4: すべてをまとめる](#step-4-put-it-all-together)

## なぜUser Agentの設定がそれほど重要なのか

[`User-Agent`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/User-Agent)ヘッダーは、HTTPリクエストを行うクライアントを識別する文字列です。通常、ブラウザ、アプリケーション、オペレーティングシステム、システムアーキテクチャに関する情報が含まれます。

たとえば、Chromeがリクエストを行う際に設定するユーザーエージェント文字列は次のとおりです。

```
Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36
```

以下は、このユーザーエージェント文字列の各コンポーネントの内訳です。

* `Mozilla/5.0`: 元々はMozillaブラウザとの互換性を示すために使われていましたが、現在では互換性目的で付与されるプレフィックスです。
* `Windows NT 10.0; Win64; x64:` オペレーティングシステム（`Windows NT 10.0`）、プラットフォーム（`Win64`）、システムアーキテクチャ（`x64`）を示します。
* `AppleWebKit/537.36`: Chromeが使用するブラウザエンジンを指します。
* (`KHTML, like Gecko`): KHTMLおよびGeckoレイアウトエンジンとの互換性を示します。
* `Chrome/127.0.0.0`: ブラウザ名とバージョンを指定します。
* `Safari/537.36`: Safariとの互換性を示します。

`User-Agent`ヘッダーは、リクエストが信頼できるブラウザから来ているのか、自動化ソフトウェアから来ているのかを判断するのに役立ちます。

Webスクレイピングボットは、デフォルトまたはブラウザではないUser Agentを使用することが多く、アンチボットシステムにとって格好の標的になりがちです。これらのシステムは`User-Agent`ヘッダーを分析し、実ユーザーとボットを区別します。[user agents for web scraping](https://brightdata.jp/blog/how-tos/user-agents-for-web-scraping-101)で詳細をご確認ください。

## Node.jsのデフォルトUser Agentとは

バージョン18以降、Node.jsには[Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)の組み込み実装として[`fetch()`](https://nodejs.org/dist/latest/docs/api/globals.html)が含まれています。外部依存なしでNode.jsでHTTPリクエストを行う推奨方法です。詳しくは[HTTP requests in Node.js with Fetch API](/blog/how-tos/fetch-api-nodejs)のガイドをご覧ください。

ほとんどのHTTPクライアントと同様に、`fetch()`はデフォルトの`User-Agent`ヘッダーを自動的に設定します。同様の挙動は[Python `requests` library](/faqs/python-requests/what-is-python-requests)でも発生します。

デフォルトでは、Node.jsの`fetch()`は次の`User-Agent`文字列を設定します。

```
node
```

`fetch()`が設定するデフォルトの`User-Agent`は、[`httpbin.io/user-agent`](https://httpbin.io/user-agent)にGETリクエストを送ることで確認できます。このエンドポイントは受信したリクエストの`User-Agent`ヘッダーを返すため、HTTPクライアントが使用しているユーザーエージェントを特定できます。

これをテストするには、Node.jsスクリプトを作成し、[`async`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)関数を定義して、`fetch()`でリクエストを送信します。

```js
async function getFetchDefaultUserAgent() {

// make an HTTP request to the HTTPBin endpoint

// to get the user agent

const response = await fetch("https://httpbin.io/user-agent");

// read the default user agent from the response

// and print it

const data = await response.json();

console.log(data);

}

getFetchDefaultUserAgent();
```

上記のJavaScriptコードを実行すると、次の文字列が返ってきます。

```
{ 'user-agent': 'node' }
```

デフォルトでは、Node.jsの`fetch()`は`User-Agent`を`node`に設定しますが、これはブラウザのユーザーエージェントとは大きく異なります。これにより、[anti-bot systems](/webinar/bot-detection)が発動する可能性があります。

アンチボットソリューションは不自然なUser Agentを検知し、そのようなリクエストをボットとしてフラグ付けしてブロックにつながります。Node.jsのデフォルト`User-Agent`を変更することは、検知回避に役立ちます。

## Fetch APIを使用してNode.jsのUser Agentを変更する方法

Fetch APIの仕様には、`User-Agent`を変更するための組み込みメソッドは含まれていません。しかし、これは単なるHTTPヘッダーであるため、[`fetch()`のヘッダーオプション](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch#setting_headers)を使って値をカスタマイズできます。

**ローカルにUser Agentを設定する**

`fetch()`は`headers`オプションでヘッダーのカスタマイズをサポートしています。次のように、特定のHTTPリクエストを行う際に`User-Agent`ヘッダーを設定できます。

```js
const response = await fetch("https://httpbin.io/user-agent", {

headers: {

"User-Agent":

"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36",

},

});
```

これらをまとめると次のようになります。

```js
async function getFetchUserAgent() {

// make an HTTP request to HTTPBin

// with a custom user agent

const response = await fetch("https://httpbin.io/user-agent", {

headers: {

"User-Agent":

"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36",

},

});

// read the default user agent from the response

// and print it

const data = await response.json();

console.log(data);

}

getFetchUserAgent();
```

上記のスクリプトを起動すると、今回は次の結果になります。

```
{

'user-agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36'

}
```

**グローバルにUser Agentを設定する**

リクエストごとに`User-Agent`を設定するのは簡単ですが、コードが繰り返しになりがちです。ただし、`fetch()` APIは現時点でデフォルト設定のグローバルな上書きをサポートしていません。

この回避策として、望む設定で`fetch()`をカスタマイズするラッパー関数を作成できます。

```js
function customFetch(url, options = {}) {

// custom headers

const customHeaders = {

"User-Agent":

"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36",

...options.headers, // merge with any other headers passed in the options

};

const mergedOptions = {

...options,

headers: customHeaders,

};

return fetch(url, mergedOptions);

}
```

これで、`fetch()`の代わりに`customFetch()`を呼び出すことで、カスタムUser AgentでHTTPリクエストを行えます。

```js
const response = await customFetch("https://httpbin.io/user-agent");
```

完全なNode.jsスクリプトは次のとおりです。

```js
function customFetch(url, options = {}) {

// add a custom user agent header

const customHeaders = {

"User-Agent":

"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36",

...options.headers, // merge with any other headers passed in the options

};

const mergedOptions = {

...options,

headers: customHeaders,

};

return fetch(url, mergedOptions);

}

async function getFetchUserAgent() {

// make an HTTP request to HTTPBin

// through the custom fetch wrapper

const response = await customFetch("https://httpbin.io/user-agent");

// read the default user agent from the response

// and print it

const data = await response.json();

console.log(data);

}

getFetchUserAgent();
```

上記のNode.jsスクリプトを起動すると、次の内容が出力されます。

```
{

'user-agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36'

}
```

## Node.jsでUser Agentローテーションを実装する

デフォルトの`User-Agent`をブラウザ文字列に置き換えるだけでは、アンチボット検知を回避できない場合があります。同じIPから同じユーザーエージェントで複数のリクエストが発生すると、アンチスクレイピングシステムはそれでも活動を自動化としてフラグ付けする可能性があります。

Node.jsで検知リスクを下げるには、リクエストにばらつきを持たせてください。効果的な手法の1つが**user agent rotation**で、リクエストごとに`User-Agent`ヘッダーを変更します。これにより、異なるブラウザから来ているように見せることができ、ブロックされる可能性を下げられます。

それでは、Node.jsでuser agent rotationを実装していきましょう。

### Step #1: User Agentのリストを取得する

[WhatIsMyBrowser.com](https://www.whatismybrowser.com/guides/the-latest-user-agent/)のようなサイトを訪問し、有効なUser Agent値をいくつかリストに追加します。

```js
const userAgents = [

"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36",

"Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:128.0) Gecko/20100101 Firefox/128.0",

"Mozilla/5.0 (Macintosh; Intel Mac OS X 14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.5 Safari/605.1.15",

"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36 Edg/126.0.2592.113",

// other user agents...

];
```

> **Tip**:
> 
> この配列に含める実世界のユーザーエージェント文字列が多いほど、アンチボット検知を回避しやすくなります。

### Step #2: User Agentをランダムに選択する

リストからユーザーエージェント文字列をランダムに選択して返す関数を作成します。

```js
function getRandomUserAgent() {

const userAgents = [

// user agents omitted for brevity...

];

// return a user agent randomly

// extracted from the list

return userAgents[Math.floor(Math.random() * userAgents.length)];

}
```

この関数で何が起きているかを分解して説明します。

* [`Math.random()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/random)は0〜1の間の乱数を生成します。
* 次に、その数値に`userAgents`配列の長さを掛けます。
* [`Math.floor()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/floor)は、結果の数値を切り下げて、その数値以下の最大の整数にします。
* 上記の処理で得られる数値は、0〜`userAgents.length - 1`の範囲のランダムなインデックスに対応します。
* そのインデックスを使って、ユーザーエージェント配列からランダムなUser Agentを返します。

`getRandomUserAgent()`関数を呼び出すたびに、おそらく異なるUser Agentが得られます。

### Step #3: ランダムなUser AgentでHTTPリクエストを行う

`fetch()`を使ってNode.jsでuser agent rotationを実装するには、`getRandomUserAgent()`関数の値で`User-Agent`ヘッダーを設定します。

```js
const response = await fetch("https://httpbin.io/user-agent", {

headers: {

"User-Agent": getRandomUserAgent(),

},

});
```

Fetch APIによって実行されるHTTPリクエストには、ランダムなUser Agentが設定されるようになります。

### Step #4: すべてをまとめる

前のステップのスニペットをNode.jsスクリプトに追加し、`fetch()`リクエストを行うロジックを`async`関数でラップします。

最終的なNode.jsのuser agent rotationスクリプトは次のようになります。

```js
function getRandomUserAgent() {

const userAgents = [

"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36",

"Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:128.0) Gecko/20100101 Firefox/128.0",

"Mozilla/5.0 (Macintosh; Intel Mac OS X 14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.5 Safari/605.1.15",

"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36 Edg/126.0.2592.113",

// other user agents...

];

// return a user agent randomly

// extracted from the list

return userAgents[Math.floor(Math.random() * userAgents.length)];

}

async function getFetchUserAgent() {

// make an HTTP request with a random user agent

const response = await fetch("https://httpbin.io/user-agent", {

headers: {

"User-Agent": getRandomUserAgent(),

},

});

// read the default user agent from the response

// and print it

const data = await response.json();

console.log(data);

}

getFetchUserAgent();
```

スクリプトを3〜4回実行してください。統計的に、以下のように異なるUser Agentレスポンスが表示されるはずです。

![different user agent responses](https://github.com/bright-jp/node-js-user-agent/blob/main/images/different-user-agent-responses-1024x298.png)

これは、user agent rotationが効果的に機能していることを示しています。

Et voilà! これで、Fetch APIを使用してNode.jsでUser Agentを設定する方法を習得できました。

## Conclusion

Node.jsでuser agent rotationを実装すると、基本的なアンチスクレイピングシステムを回避するのに役立ちます。しかし、より高度なシステムでは自動化されたリクエストが検知・ブロックされる可能性があります。IP banを回避するには、IPおよびUser Agentローテーションなどの機能を通じてアンチスクレイピング対策を効果的にバイパスし、Webスクレイピングをこれまで以上に容易にする[Web Scraper API](https://brightdata.jp/products/web-scraper)の利用をご検討ください。

今すぐ登録して、無料トライアルを開始しましょう！