---
cover: ./webpack-config-fragment.png
date: 2016-12-23T20:00:00+09:00
lastmod: 2016-12-25T12:45:00+09:00
title: import()を使ったHTTP/2時代のフロントエンド実装
---

この記事は[http2 Advent Calendar 2016](http://qiita.com/advent-calendar/2016/http2)の二十三日目の記事である。

今年……2016年は[HTTP/2](https://http2.github.io/)が大いなる躍進を遂げた年であったと感じる。

[Amazon CloudFront](https://aws.amazon.com/jp/cloudfront/)も今年の九月にHTTP/2の対応が追加された ([Amazon CloudFront now supports HTTP/2](https://aws.amazon.com/about-aws/whats-new/2016/09/amazon-cloudfront-now-supports-http2/))。また[Fastly](https://www.fastly.com/)でも十一月から一般利用が可能となった ([HTTP/2 is now in General Availability](https://www.fastly.com/blog/http2-now-general-availability))。

これまでは[Nginx](https://nginx.org/)や[Apache HTTP Server](https://httpd.apache.org)などのHTTP/2に対応したHTTPサーバーをフロントとして自身で運用しなければならなかった。Contents Delivery Network (CDN) サービスの恩恵を享受できず、また運用の手間をかける必要もあった。

しかし今年 多くのCDNサービスがHTTP/2に対応したため、手間をかけることなく誰もがHTTP/2の恩恵を気軽に享受できるようになった。今後HTTP/2で配信されるウェブページが増えていくことであろう。

<!--more-->

HTTP/2が策定される以前のHTTPの規格であるHTTP/1.1では同一のホスト名に対して同時にリクエストできる数には制限がある。そのため、複数のスクリプトファイルが埋め込まれたウェブページはページ全体の読み込みが完了するまでに時間がかかってしまっていた。それを解決するために複数のスクリプトファイルを[webpack](https://webpack.js.org/)や[Browserify](http://browserify.org/)といったバンドルツールを用いて結合させるといったことが、半ば当たり前のこととなっていた。

HTTP/2ではリクエストとレスポンスのストリームが多重化されている。並列して複数のリクエストを送信できるようになったため、これまで当たり前のことであったスクリプトファイルの結合の必要性は薄くなった。

```jsx
// src/client.jsx

import React from 'react';
import ReactDOM from 'react-dom';

function render(element, container) {
  return new Promise((resolve, reject) => {
    try {
      ReactDOM.render(element, container, resolve);
    } catch (error) {
      reject(error);
    }
  });
}

async function main() {
  const App = await import('./components/App');
  const container = document.getElementById('root');
  try {
    await render(<App />, container);
  } catch (error) {
    // eslint-disable-next-line no-alert
    alert(error.message);
  }
}

main();
```

のように[`import()`](https://github.com/tc39/proposal-dynamic-import)で別のスクリプトファイルを指定すると実行時にそのスクリプトファイルが読み込まれるようになる。`import()`は`Promise`を返すので、ECMAScript 2017から仕様に追加される[Async Functions](https://tc39.github.io/ecmascript-asyncawait/)との相性が極めて良い。

`import()`はこの記事を執筆している時点 (2016年十二月二十三日) ではstage 3であり、正式な仕様となっているものではない。そして実装されているウェブブラウザーもまだ存在していない。しかしwebpack (v2.1.0-beta.28以降) を使うことによって、期待するような効果が得られる。

webpack単体でも`import()`に対応しているが、Babelも一緒に使うのであればBabelで処理させるために[`babel-plugin-syntax-dynamic-import`](https://www.npmjs.com/package/babel-plugin-syntax-dynamic-import)というBabelのプラグインを導入させなければならない。

またBabelは[CommonJS](http://www.commonjs.org/)に対応していないので、webpackから読み込んでも実体はそのまま返されない。`default`という名前のプロパティーを含む`object`が返されるため、`(await import('./components/App)).default`のような形で呼び出さなければならず、いささか煩雑である。[`babel-plugin-add-module-exports`](https://www.npmjs.com/package/babel-plugin-add-module-exports)を使うことによって解決できる。`babel-plugin-add-module-exports`はその名前の通り、`module.exports`を追加するBabelのプラグインである。これによりwebpackの想定するCommonJSに対応する形へのトランスパイルが可能となる。

最終的にwebpackの設定 (`webpack.config.js`) は次のようになる。

```javascript
// webpack.config.js

const BabiliPlugin = require('babili-webpack-plugin');
const HtmlPlugin = require('html-webpack-plugin');
const path = require('path');
const merge = require('webpack-merge');

const clientConfig = {
  devtool: 'source-map',
  entry: {
    main: path.join(__dirname, 'src', 'client.jsx'),
  },
  module: {
    rules: [
      {
        exclude: /node_modules/,
        options: {
          babelrc: false,
          plugins: [
            'add-module-exports',
            'syntax-dynamic-import',
          ],
          presets: [
            ['env', {
              debug: true,
              targets: {
                browsers: [
                  'Chrome &gt;= 55',
                ],
              },
            }],
            'react',
          ],
        },
        test: /\.jsx?$/,
        loader: 'babel-loader',
      },
    ],
  },
  output: {
    chunkFilename: '[id].[chunkhash].js',
    crossOriginLoading: 'anonymous',
    filename: '[name].[chunkhash].js',
    path: path.join(__dirname, 'build', 'public'),
    publicPath: '/',
  },
  plugins: [
    new HtmlPlugin({
      template: path.join(__dirname, 'src', 'templates', 'index.ejs'),
    }),
  ],
  resolve: {
    extensions: [
      '.js',
      '.jsx',
    ],
  },
};

module.exports = (env) => {
  switch (env) {
    case 'production':
      return merge(clientConfig, {
        plugins: [
          new BabiliPlugin(),
        ],
      });
    default:
      return merge(clientConfig, {
        output: {
          chunkFilename: '[id].js?[chunkhash]',
          filename: '[name].js?[chunkhash]',
        },
      });
  }
};
```

今回 書いたコードの全体は[ykzts-sandbox/try-dynamic-import](https://github.com/ykzts-sandbox/try-dynamic-import)にある。[MITライセンス](https://opensource.org/licenses/MIT)で公開しているので自由に使ってもらって構わない。

今後、HTTP/2での通信はより一般的なものになっていくと予想される。HTTP/1.1時代の常識のままでいては良くないだろう。フロントエンドに携わるソフトウェア開発者でも、HTTPについてある程度の知識と理解を持ち、よりユーザーの方向を向いた適切な開発を心がけて行きたい。
