---
title: "SPAのダウンタイムなしリリース"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react","SPA","devops"]
published: false
---


## 経緯

### なぜダウンタイム無し目指すのか

一つ大事な要因として、toBのアプリケーションなので、停止リリースするにはお客様の利用時間を避ける必要があり、それは通常深夜または早朝になりがちです。さらにお客様が増えると、この深夜早朝に仮に利用するお客様がいると、リリースするタイミングすら見つからなくなる恐れがあります。

アプリケーションのリリースからまださほど経っていないので、早いうちでダウンタイム無しでリリースを達成しないと、今後の開発サイクルに大きな支障が出るかねません。バックエンドのアプリケーションのリリースは、以前紹介した[ CloudRun Service + Jobsのコンビネーション ](https://zenn.dev/optimind/articles/7fefd01bbd8905)で達成できていますが、今回はフロントエンドのアプリケーションをダウンタイムなしにリリースするのが目的です。


### 現象

SPAでユーザー影響ないリリース、いわばゼロダウンタイムのリリース（厳密には100%一致するわけではないが）するには、思ったより問題が厄介でした。

何かというと、こちらの[スレッド](https://github.com/vitejs/vite/issues/11804)にもあったように 、ユーザーが利用中にリリースしてしまうと、下記のエラーになってページが表示できなくなる可能性があります。

```
TypeError: Failed to fetch dynamically imported module
```


### 環境

さて、この現象が起こっていた、フロントエンドのアプリケーションの環境を紹介します。


```json
{
    "node": "20.11.1",
    "vite": "4.3.9",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "@tanstack/react-query": "^4.28.0",
    "@trpc/client": "^10.21.0",
    "@trpc/react-query": "^10.21.0",
    ...
}
```

なお、フロントエンドのアプリケーションは、Firebase Hostingにホストされています。ここはキャッシュ周りの設定が絡んでいるのですが、内容に関しては他のホスティングサービスにも参考できると思います。

## 原因

[こちら](https://github.com/vitejs/vite/issues/11804)のスレッドではすでに述べられていますが、このエラー発生の流れとして、

- verAのアプリをデプロイした
- ユーザーがverAを利用中
- コードを変更して、verBをデプロイした
- ユーザーが画面遷移をする
- `Failed to fetch dynamically imported module`エラーになる
- 画面をリロードしてもエラーは消えない

ここでいくつかの要素をまず整理したいと思います。

- chunk化とlazy load: 一つ大きなファイルより、小さく分割して、必要な時だけファイルを都度取得する方針となります。vite(rollup)はもちろん、ほとんどのビルドツールはファイルをchunk化しています。ユーザーが画面を開く時に、遷移先もしくは現在画面上に表示する必要のないものがあれば、一旦ファイルの名前などを保持して、**必要となった時にまたリクエストする**ようになります。
- 命名のハッシュ値: ファイルのコンテンツからハッシュ値をファイル名につけることです(e.g. `Component.abc.js`)。コードを変えるとビルド後のファイル名のハッシュ値も変わります(e.g. `Component.def.js`)。ただ、これは単にファイルに変更があるかどうかがわかるだけではなく、**キャッシュをすべきかどうかの指針**にもなってくれます。
- リリースのタイミング: 今回のケースは、利用中に新しいバージョンがリリースされたら、とのタイミングも重要です。**利用中にリリースされると、ファイル名のハッシュ値変更により、古いファイルがなくなる**のです。ここで画面遷移すると問題が発生します。
- ブラウザーキャッシング: このエラーが発生した後、画面をリロードしても同じ問題になる可能性があります。ファイルが存在しない時に、ホスティングサーバーからなにかしらのフォールバックファイルが返ることがよくあるパターンです。その**ファイルがブラウザーにキャッシュされると、キャッシュ有効時間が切れるまでリロードしても消えない**のです（もしくはdevtoolからdisable cacheにチェックしてリロードする）。

すると、解決の方向性もなんとなくわかってきます。

- 画面ロードの効率化や、キャッシュ判定などの目的から、ファイルはchunk化され、かつ内容変更するたびにビルド後のファイル名が変わる前提にしたい([ ハッシュを抜いてファイル名を変更しない ](https://stackoverflow.com/questions/69614671/vite-without-hash-in-filename)ようにするのは邪道とみなす)
- 新しいバージョンがリリースされた、という情報をアプリ側で持つようにしたい
- 新しいバージョンを検知したら、新しいバージョンへスムーズに移行できるようにしたい
- もしくは、古いバージョンでもしばらく動けるようにしたい
- ファイルがブラウザーにキャッシュされないように、キャッシュやフォールバックファイルなど設定を調整したい


## 解決トライ１ーService Worker

これは真っ先にきました。よく他のアプリで見られたりしますが、「アップデートがあります。クリックして更新してください」的なトーストメッセージが右下に出てきて、それでクリックしたら画面がリロードして新しいバージョンになります（[ 参考記事 ](https://medium.com/progressive-web-apps/pwa-create-a-new-update-available-notification-using-service-workers-18be9168d717)）。それで早速調査と実験を始めました。結論から言うと、解決には至らなかったが、良き経験と学びになりました。

### 背景知識

まずサービスワーカーとは何かを軽くまとめます。

- client（ブラウザー）とserver側に介在する代理/proxy
  - clientからのリクエストをインターセプトする
  - clientのmain threadへ通知をプッシュすることが可能 -> ここ重要
  - serverへリクエストを送る
  - オフライン時のキャッシング
- ワーカーのコンテキストで実行されているため、DOMへのアクセスはない
  - とはいえ、serviceWorkerとmain threadの間に通信が可能なため、メッセージによってDOM操作や通知トーストなどを含める画面上の操作が可能
- ブラウザーのサブプロセスとして「バックグラウンド」で実行される（スレッドではなく）
  - SWのコードはブロックされない
  - ページのアプリコードのコードもブロックしない（画面レンダーなどを担うmain threadから完全独立）
  - レジスターされたら複数の画面（タブ）で利用でき、タブがクローズされてもなくならない
- httpsまたはlocalhostしか使えない

サービスワーカーの[ ライフサイクル ](https://developer.chrome.com/docs/workbox/service-worker-lifecycle)や詳しい使い方はここで割愛しますが、一言に言えば、**アプリケーションのレンダリングなどを担当するメインスレッドと通信可能なバックグラウンドプロセスであり、ブラウザーとサーバーの間に介在する代理である**、と認識があれば十分かと。


### どう解決するか

SWでどう言うふうにこの問題を解決できるかと言うと、主に**新しいバージョンの検知**に期待しています。具体的には、

- service workerのupdatefoundイベントを利用して、service workerのファイルに変更があることがわかる
- service workerファイルの更新自体は、アプリコードと関係ないが、リリースのビルド度にコミットハッシュが変わるため、そのハッシュ値をservice workerのファイルにインジェクトすると、更新のトリガーとしては成立する
- 更新があると分かったら、service workerから画面へメッセージを送り、アプリコードで更新の通知を出す

更新の通知を出すかどうかはUIUXの領域で議論する余地はまだありますが、ユーザーに動きを案内するより、完全に裏側で黙々アップデートする方法も存在します（[参考記事](https://dev.to/atonchev/flawless-and-silent-upgrade-of-the-service-worker-2o95)）。いずれにしても、サービスワーカー経由で新しいバージョンの検知が達成できれば、後は簡単になります。


### ブラウザーバージョンと対応状況

API自体はそれほどに新しいわけではないが、念の為バージョンをメモしておいた方が良い（2024/06/30時点）

![](https://storage.googleapis.com/zenn-user-upload/970336acf905-20240630.png)

### ServiceWorkerをReactへ導入

ここでは主に2つのルートがあります。

一つは[ Viteのプラグイン ](https://vite-pwa-org.netlify.app/guide/register-service-worker)（中身は[ workbox ](https://github.com/GoogleChrome/workbox)なので、直接workboxを使うのも同じルートとしてみなす）。React用の[ アップデート通知のコンポーネント ](https://vite-pwa-org.netlify.app/frameworks/react.html#prompt-for-update)まで用意してくれていますので、手取り早く実装したい場合はこちら断然おすすめです。

もう一つはVanilla JS。今回は勉強もかねてこちらにしました（と言うのは建前で、実装が終わってからviteのプラグインを知りました）。主に[こちらの記事](https://medium.com/@foyemc/implementation-of-service-worker-using-reactjs-application-to-build-pwa-6366fd9a0527)を参考にして実装しています。

:::details sw.js
```js
// sw.js -> サービスワーカーの本体
// injected by vite during build
const currentVersion = '<commit-hash>';

self.addEventListener('install', (e) => {
  console.log('Service worker install', e);
  self.skipWaiting();
});

self.addEventListener('activate', async (e) => {
  // Take control of the page as soon as the service worker is activated.
  self.clients.claim().then(() => {
    self.clients.matchAll().then((clients) => {
      for (const client of clients) {
        client.postMessage('service worker activated');
      }
    });
  });
});

self.addEventListener('message', (e) => {
  // NOTE: main threadから通知が受け取ると、全て開いているタブに`UPDATE_AVAILABLE`を送信し、
  // 更新があるとの状態はわかるようになる
  if (e.data && e.data.action === 'notifyUpdate') {
    console.log('remove cache');
    e.waitUntil(
      caches
        .keys()
        .then((cacheNames) => {
          return Promise.all(
            cacheNames.map((cache) => {
              // if (!cacheWhitelist.includes(cache)) {
              return caches.delete(cache);
              // }
            }),
          );
        })
        .then(() => {
          console.log('notify main thread for update');
          self.clients.matchAll().then((clients) => {
            for (const client of clients) {
              // Post a message to each client to reload the page
              client.postMessage({ type: 'UPDATE_AVAILABLE' });
            }
          });
        }),
    );
  }
});
```
:::

さらに、通信する側のメインスレッドでレジスターするのと、イベントリスナーを追加するのが必要です。

:::details registerServiceWorker
```ts
const registerServiceWorker = () => {
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker
      .register('/sw.js', { scope: './' })
      .then(function (registration) {
        console.log('Service worker registered', registration);
        // NOTE: updatefoundイベントを監視して、発火時にサービスワーカーに通知を出す
        registration.addEventListener('updatefound', () => {
          const newServiceWorker = registration.installing;
          if (newServiceWorker != null) {
            newServiceWorker.addEventListener('statechange', () => {
              if (newServiceWorker.state === 'activated') {
                console.log('update available');
                newServiceWorker.postMessage({ action: 'notifyUpdate' });
              }
            });
          }
        });
      })
      .catch(function (error) {
        console.log('Failed to register service worker', error);
      });
  }
};
```
:::

上記の関数を、Reactのルートコンポーネントをレンダリングする際に実行します。

:::details root
```tsx
const rootElement = document.getElementById('root')!;
if (rootElement.innerHTML === '') {
  ReactDOM.createRoot(rootElement).render(
    <React.StrictMode>
      <AppRoot />
    </React.StrictMode>,
  );
  registerServiceWorker();
}
```
:::

あとは、アップデートの情報を渡すために、コンテキストプロバイダーを作ります。

:::details ContextProvider
```tsx
function AppUpdateProvider({ children }) {
  const [hasNewAppVersion, setHasNewAppVersion] = useState(false);

  useEffect(() => {
    if ('serviceWorker' in navigator) {
      const handleMessage = (e: MessageEvent) => {
        if (e.data?.type === 'UPDATE_AVAILABLE') {
          console.log('Update available!');
          setHasNewAppVersion(true);
        }
      };

      navigator.serviceWorker.addEventListener('message', handleMessage);

      // Cleanup listener on unmount
      return () => {
        navigator.serviceWorker.removeEventListener('message', handleMessage);
      };
    }
  }, []);

  return (
    <AppUpdateContext.Provider value={{ hasNewAppVersion }}>
      {children}
    </AppUpdateContext.Provider>
  );
}
```
:::


これで、新しいバージョンがあるかどうかの情報は、アプリ側で知ることができるようになります。そのあとはトースト通知なり、自動リロードなりが可能になります。

### SW案の問題点

これでまで見ると、**更新検知**が可能になったように見えてしまいますが、実はかなり致命的な問題があります。

- 新しいバージョンを検知する、`updatefound`イベント自体は頻繁に起こらない
  - 定期更新検知は、24時間置きになる -> 長すぎて解決にならない
  - もしくは画面ロード・リロード時に更新チェックが行われる -> 鳥が先か卵が先かの問題になる
- 自動更新検知を諦め、手動で更新検知する（[update](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerRegistration/update)でポーリング）と、一定の頻度を超えると`Service workder self-update limit exceeded`のエラーが起こる


また、もう一つ潜在的な複雑度を上げる問題として、サービスワーカーにはキャッシュができるが、ブラウザーのHTTPキャッシュとは別系統かつブラウザーのHTTPキャッシュはコードで制御できないため、うまくキャッシングするために、ブラウザーのHTTPキャッシュを無効にして、**全部サービスワーカー側でキャッシュ管理する**ことになってしまいます。

キャッシュ制御関連の実験を始める前に、すでにこの方針の限界を感じてしまい、舵を切るように決めました。

## 解決トライ２ーポーリングとルート遷移監視

上記の案では躓いてしまいましたが、良いヒントが実はありました。要は、**新しいバージョン検知は、より短い頻度でポーリングするのであれば、アプリ側でもできる**とのことです。

### バージョンファイル

SW案では、`sw.js`ファイルにコミットハッシュをビルド時にインジェクトすることで、新しいサービスワーカーを作っています。

このコミットハッシュのバージョン情報は、`version.json`とかのファイルに置いておけば、ブラウザーからそのファイルをフェッチすればよいのです。

:::details version.json
```json
{"commitHash": "54f83fdfc8bb9935bd268b7ef3f869ce9d931a5d"}
```
:::

VITEでビルド時にバージョンファイルを作る例として、

:::details vite.config.ts
```ts
import react from '@vitejs/plugin-react-swc';
import { execFileSync } from 'node:child_process';
import { mkdirSync, writeFileSync } from 'node:fs';
import path from 'node:path';
import { defineConfig, loadEnv, type PluginOption } from 'vite';

// https://vitejs.dev/config/
export default defineConfig(({ mode }) => {
  // ...
  const commitHash = execFileSync('git', ['rev-parse', 'HEAD']).toString().trim();
  const versionInfo: PluginOption = {
    name: 'version-info',
    apply: 'build',
    generateBundle() {
      const distDir = path.resolve(__dirname, 'dist');
      const filePath = path.join(distDir, 'version.json');
      // 実行順番次第でdistフォルダーが作られていない可能性がある
      mkdirSync(distDir, { recursive: true });
      writeFileSync(filePath, JSON.stringify({ commitHash }));
    },
  };

  return {
    define: {
        // FE側でグローバル定数としてアクセス可能になる
      __VERSION_COMMIT_HASH__: JSON.stringify(commitHash),
    },
    plugins: [versionInfo, react()],
    // ...
  };
});
```
:::

これで都度`version.json`ファイルがビルド先のフォルダにおかれるようになり、アプリ側のコードにもグローバルの定数としてバージョンの情報がアクセスできるようになります。

次はアプリ側でその内容をポーリングすれば良いのです。

:::details useAppVersion.ts
```ts
const useAppVersion = () => {
  const { isSuccess, data } = useQuery<{ commitHash: string }>({
    queryKey: ['appVersion'],
    queryFn: () => fetch('/version.json').then((r) => r.json()),
    refetchInterval: 60 * 1000,
    // ...
  });
  return isSuccess ? data.commitHash : null;
};
```
:::

### ルート遷移監視

これまで更新検知をポーリングの形で可能にしました。更新がわかったあとの動きとして、今回の戦略は、通知トーストまたは黙々アップデートではなく、プランCにしました。このエラー自体は、新しいリリースがあり、ルート遷移する際に起こるため、**ルート遷移を検知できれば、自動で画面ロードを行う**ことで、新しいバージョンのファイルを取得できるようになるはずです。

実装方法はFE側で利用するルーティングライブラリーによりますが、考え方は同じはずなので、ここは[reactrouterの例](https://reactrouter.com/en/main/start/concepts#history-object)を出します


:::details AppVersionWrapper
```ts
// import { useLocation } from 'react-router-dom'; -> も一応ルート監視可能
function AppVersionWrapper({ children, history }) {
  const remoteVersion = useAppVersion();
  const localVersion = __VERSION_COMMIT_HASH__;
  const hasNewAppVersion = useMemo(() => {
    return (
      remoteVersion != null &&
      remoteVersion !== localVersion
    );
  }, [remoteVersion, localVersion]);

  const registered = useRef(false);
  useEffect(() => {
    if (!hasNewAppVersion) return;
    if (registered.current) return;
    registered.current = true;
    const unlisten = history.listen(() => {
      console.log('new app version detected, reloading...');
      window.location.href = history.location.href;
    });
    return () => {
      unlisten();
      registered.current = false;
    };
  }, [hasNewAppVersion, router]);

  return <>{children}</>;
}
```
:::

### まとめ

このトライでできたこととして

- アプリケーションのバージョン情報（コミットハッシュ）をアプリ側とホスティングサーバーに両方保持させる
- アプリ側ではホスティングサーバーのバージョン情報に対してポーリングすることで、更新検知ができるようになる
- 更新検知ができると、その情報をフロントエンドのステートとして保持する
- ユーザーのルート遷移を監視し、更新がある際に画面ロードを行う

## 解決トライ3ーキャッシュとリダイレクト戦略調整

上記のトライでは、アプリ更新のバージョンポーリングと更新検知解決になっていますが、まだ一つ大きな問題が残っています。つまり、**画面リロードしても、エラーが消えない可能性がある**ところです。

これは原因分析のところで触れましたが、ブラウザーキャッシュによって問題のあるファイルをキャッシュしてしまい、TTLが切れるまでエラーが消えないのです。`Failed to fetch dynamically imported module`のエラートレースを見ると、MIME`タイプのエラーが起きているのです。トライ2のルート遷移監視は、厳密に言えば、`beforeRouteChanged`ではなく、`afterRouteChanged`になるので、存在しないファイルのリクエストが先に飛ばされると、この問題に繋がります。

```
Failed to load module script: Expected a JavaScript module script but the server responded with a MIME type of "text/html". Stict MIME type checking is enforced for module scripts per HTML spec.
```


そのため、ここの考えとしては、

- **キャッシュしたいファイルと、キャッシュしたくないファイルを整理する**
- なんでも`index.html`でフォールバック・リダイレクトするのではなく、**`index.html`で返って良いパターンと、そうしたくないパターンを整理する**
 - **`index.html`にしたくないパターンの場合の代案を決める**

アプリケーションはfirebase hostingにホストされているため、以下はfirebase hosting用の設定で例にします。

### キャッシュ対象整理

調整後の`headers`は以下となります。

:::details firebase.json
```json
{
  "hosting": {
    "headers": [
      {
        "source": "/**",
        "headers": [
          { "key": "Cache-Control", "value": "no-cache,no-store" },
        ]
      },
      {
        "source": "**/*.@(jpg|jpeg|gif|ico|png|svg|webp|js|css|eot|otf|ttf|ttc|woff|woff2|font.css)",
        "headers": [
          { "key": "Cache-Control", "value": "max-age=31536000,immutable" },
        ]
      }
    ]
  }
}
```
:::

ここのポイントとして
- `js`, `css`や画像、フォントなどの静的ファイルはキャッシュしたい
  - 他のファイルはビルド時のファイル名ハッシュによるバージョンニングあるので、`immutable`で良さそう（[ 参考 ](https://simonhearne.com/2022/caching-header-best-practices/#the-solution)
  - ただ、`immutable`がブラウザーにサポートされていない可能性があるので（[参考](https://caniuse.com/mdn-http_headers_cache-control_immutable)）、高い`max-age`(上記では1年)で無難
- 他のファイルというと、`index.html`と`version.json`に当たる
  - この二つのファイルはキャッシュして欲しくないので、`no-cache`か結構低い`max-age`で良い（[ 参考 ](https://stackoverflow.com/questions/48589821/firebase-hosting-how-to-prevent-caching-for-the-index-html-of-an-spa)）
  - ただ、低い`max-age`だと、やはりその分のキャッシュ時間が発生するので、徹底的に解決するには基本`no-cache`にしたい

なお、firebase hostingではデフォルトでCDNにファイルをキャッシュしていますが、新しいデプロイがあると、CDN上のキュアッシュは[パージされる](https://zenn.dev/teramotodaiki/articles/676468e8e40763)ので、そこの制御は特に考慮していません。

### `index.html`でフォールバックするパターン

`index.html`でフォールバックするパターンは以下となります([rewritesについての参考](https://firebase.google.com/docs/hosting/full-config#rewrites))。

:::details firebase.json
```json
{
  "hosting": {
    "rewrites": [
      {
        "source": "/",
        "destination": "/index.html"
      },
      {
        "source": "/user{,/**}",
        "destination": "/index.html"
      },
      {
        "source": "/admin{,/**}",
        "destination": "/index.html"
      }
    ]
  }
}
```

ここのポイントとして、
- アプリ側存在するURLのパターン＝画面・ページのあるパターンだけをフォールバック対象とする
  - `/`, `/user`から始まるURL、`/admin`から始まるURL、という3つのパターンのみ、`index.html`がフォールバックになる
  - 上記以外のURLにアクセスすると、**404エラーになる**
- js, cssなどの静的ファイルはフォールバックしない
  - 静的ファイルのリソースURIとして、`/abc.js`, `/def.css`になっている
  - 画面遷移先には通常`js`ファイルが求められているため、`/abc.js`が求められる時には`index.html`が返らず、**404エラーになる**



### 404の場合ー画面

さて、フォールバックしたくない場合は整理できたところで、結局404はどうするの、との問題が残ります。

ここでまずは、存在しないURLにアクセスする時の404について対策をまとめます。

他のホスティングサービスも共通すると思いますが、404の場合は、もし`404.html`のファイルが提供されていなかったら、デフォルトではfirebaseの404画面が見えてしまいます。

rewritesで、`404.html` -> `index.html`との発想も一応ありました。つまり、404のページをあえて設けず、ホームページを代わりにする作戦です。ただ、ルーターにとって、404エラーになるURLが、どのパターンにもマッチしないので、レンダリングするコンポーネントもなく、画面が真っ白のままになってしまいます。

結局一番素朴な方法にして、404.htmlファイルを一つ作りました。

なお、このファイルを正しくビルドするために、`vite`の設定も少し変更が必要です。

:::details vite.config.ts
```ts
export default defineConfig(({ mode }) => {
  // ...

  return {
    define: {
        // FE側でグローバル定数としてアクセス可能になる
      __VERSION_COMMIT_HASH__: JSON.stringify(commitHash),
    },
    plugins: [versionInfo, react()],
    build: {
      rollupOptions: {
        input: {
          main: path.resolve(__dirname, 'index.html'),
          404: path.resolve(__dirname, '404.html'),
        },
      },
    }
  };
});
```
:::

### 404の場合ー静的ファイル

上記で存在しない画面の404フォールバックが解決できましたが、静的ファイルのリソースを取得する際にの404エラーについて、エラー捕獲用のコンポーネントを用意します。

:::details AppErrorComponent.tsx
```tsx
const isModuleNotFoundError = (error: unknown): boolean => {
  return (
    error instanceof Error &&
    typeof error.message === 'string' &&
    error.message.includes('Failed to fetch dynamically imported module')
  );
};

function AppErrorComponent({ error }) {
  const isProd = import.meta.env.NODE_ENV === 'production';
  useEffect(() => {
    console.error('An error occurred', error);
    if (isModuleNotFoundError(error)) {
      console.log('reloading...');
      // prod環境以外はデバッグのために画面リロードしない
      if (isProd) {
        window.location.reload();
      }
    }
  }, [error, isProd]);

  // 特になにもレンダリングしない
  return <CustomErrorComponent> {...} </CustomErrorComponent>;
}
```
:::

ここのポイントとして、エラーを捕獲したところで、ユーザーにエラーの画面を見せず、何もレンダーリングしないまま、リロードを行うところです。

リロードを行うことによって、`index.html`がキャッシュされないため、新しいバージョンが取得できるようになり、エラーはユーザーに見せずに済むことです。

### まとめ

このトライでできたこととして、ホスティングサーバー側の設定調整し、
- キャッシュしたいファイルと、キャッシュしたくないファイルを切り分けられた
- `index.html`で返って良いパターンと、そうしたくないパターンを切り分けられた
- `index.html`にしたくないパターンの場合は
  - 存在しない画面に対して404.htmlファイルを提供する
  - 存在しない静的ファイルに対してエラーコンポーネントを提供する

これらの対策によって、この問題はほぼ解決に近いとも言えるようになりました。

## 解決トライ4ー複数バージョンのファイルを提供

これまで問題は解決されているはずですが、万が一を防ぐために、実は別の対策も打っていました。というのは、更新検知のポーリングは、1分間隔で起きるため、どうしてもタイミングが悪く、その間隔中にリリースがあって、検知できず例のエラーになることがあり得ます。

この問題の根源で言えば、新しいバージョンのリリースによって、**必要なファイルがなくなっている**ところにあるので、逆に考えると、**古いファイルを新しいリリースによって削除されないようにすれば良い**、とも考えられます。

もちろん、時にバグ修正などの関係で古いファイルは削除したいのですが、そのケースも備えて対処したいところです。

ただ、firebase hostingにはそういう機能がないため、ここで複数のリリースのファイルを同時に提供する方法をまとめます。

### 設計

複数のバージョンのファイルを提供するためには、firebase hostingへビルド後のファイルアップロードする際に、**以前のファイルを含めてアップロード**すれば良いことです。

問題として、
- ファイルはどこで、どういう構造で保存するか
- どこまでのファイルを含めるか、リリースが積み重なってファイルが多くなった時にどうすれば良いか

一つ目の問題について、リリース毎にファイルを保存する場所が必要で、今回はGCSのバケットにしました。

- 環境毎にバケットを作り、liveとactiveの2つのフォルダーを設ける
- liveフォルダーには、firebase hostingと同じファイルを保存する
  - ここで複数リリースバージョンのファイルが存在すると想定
- activeフォルダーには、今までアップロードされた各バージョンを独立のフォルダーに保存する
  - 保存先は`active/<YYYY-MM-DD>/<timestamp>-<commit-hash>/<file-path>.<ext>`の形に固める
  - ソートしやすくするために、日付とタイムスタンプを先頭につける

二つ目の問題について多少複雑になりますが、下記の3つのAPIで対処するようにしました。

- `upload`: 現在のコミットに基づいてビルドされたファイルをストレージバケットactiveフォルダーにアップロードする
- `merge`: 引数で決められた複数のバージョンのファイルをダウンロードし、一つのフォルダーにマージし、liveフォルダーを上書きする
- `purge`: 引数で決められたバージョンをストレージバケットactiveフォルダーから削除する

リリース自体は全部CDワークフローになりますが、想定するフローとして、
- `npm run build`で現在のコミットハッシュで、`dist`フォルダを作成
- `upload`で上記の`dist`をユニークバージョンとして`active`フォルダーにアップロードする
- `merge`で決められた`active`内のバージョンをダウンロードして、実行環境で`dist`フォルダーにマージさせ、`live`フォルダーを上書きする
- 上記の`dist`フォルダーを用いて、firebase hostingにデプロイする
- 後処理として、一定期間より古いバージョンを`purge`で削除する
  - `purge`は定期実行やファイルのライフサイクルで自動削除も検討しましたが、結局リリースの頻度は基本二週間1-2回程度で、リリースの後で実行すれば十分と決めました。


### 実装

これらのAPIをどこから実行するのか、との議論もありますが、今回のプロジェクトでは、`oclif`を利用してCLIコマンドを作成していたため、そのまま踏襲してコマンドを作成しました。ワークフロー実行中は、こららのCLIコマンドを実行するだけで目的は達成できます。

細かい実装を省いて、インターフェースと一部要注意な部分のみ提示します。

#### `base`

コマンド共通の部分を定義します。これはあくまでも継承目的であって、直接コマンドとして使いません。

:::details base
```ts
import type { Bucket, File } from '@google-cloud/storage';
import { Storage } from '@google-cloud/storage';
import { Command, Flags } from '@oclif/core';

export default abstract class Base extends Command {
  static description =
    'multi-release base command, this is only an abstract class for inheritance purpose';

  static hidden = true;
  static flags = {
    bucket: Flags.string({
      description: 'bucket name',
    }),
    env: Flags.enum({
      description: 'environment name',
      options: ['prd', 'stg', 'dev'],
      required: true,
    }),
  };

  abstract commandName: string;

  run(): Promise<void> {
    throw new Error('Invalid operation, command is not executable!');
  }

  getBucketName(env: Env): string;

  getProjectId(env: Env): string;

  getLocalDistDir(): string;

  findReleases(files: readonly string[]): string[];

  async listAllFiles(client: Bucket) {
    const files: string[] = [];
    return await new Promise<string[]>((resolve, reject) => {
      // NOTE: ここは要注意、実装当時では、フォルダーのみ取得するのはできなかった
      // {autoPaginate:false, dilimiter: '/'}でも正しいパスは取れない
      client
        .getFilesStream({ prefix: 'active/' })
        .on('error', reject)
        .on('data', (file: File) => {
          files.push(file.name);
        })
        .on('end', () => {
          resolve(files);
        });
    });
  }

  async uploadFilesInDir(client: Bucket, localDir: string, destDir: string): Promise<number>;

  async getStorageClient(projectId: string, bucket: string): Promise<Bucket>;
}
```
:::

#### `upload`

基本継承するメソッドを使えば完成。

:::details upload
```ts
import { Flags } from '@oclif/core';
import Base from './base';

export default class Upload extends Base {
  static description =
    'multi-release upload command, upload current build files to remote storage bucket';

  static examples = [
    `$ app-cli multi-release upload --bucket=...  --partition=... --commit=...`,
  ];

  static flags = {
    ...Base.flags,
    commit: Flags.string({
      description: 'commit hash of current release',
      required: true,
    }),
  };

  commandName = 'upload';

  async run(): Promise<void> {
    const {
      flags: { bucket, env, commit },
    } = await this.parse(Upload);

    const bucketName = bucket ?? this.getBucketName(env);
    const projectId = this.getProjectId(env);
    const client = await this.getStorageClient(projectId, bucketName);
    const destDir = this.getUploadDestDir(commit);
    try {
      const localDistDir = this.getLocalDistDir();
      const count = await this.uploadFilesInDir(client, localDistDir, destDir);
      // ... log
    } catch (error: unknown) {
      // 失敗時に該当リリースフォルダーを削除する
      await client.deleteFiles({ prefix: destDir });
      throw error;
    }
  }

  // Format: active/<YYYY-MM-DD>/<timestamp>-<commit>/
  getUploadDestDir(commit: string): string;
}
```
:::

#### `merge`

リリースをどのように見つけるか、方針は2つ決めました。日数（例えば7日以内）と回数（例えば直近３バージョン）が両方受け取ることができますが、同時にある場合は、よりリリース件数の多い方が優先されます。

:::details merge
```ts
import { type Bucket } from '@google-cloud/storage';
import { Flags } from '@oclif/core';
import Base from './base';

export default class Merge extends Base {
  static description =
    'multi-release merge command, fetch multiple build version files and merge them into a single build, upload to remote storage bucket';

  static examples = [
    `$ app-cli multi-release merge --bucket=... --days=... --partition=... --latest=2`,
  ];

  static flags = {
    ...Base.flags,
    days: Flags.integer({
      description:
        'keep releases in last n days, the larger value will be prioritized if conflicted with `latest`',
      min: 1,
      max: 10_000,
    }),
    latest: Flags.integer({
      description:
        'keep releases in last n versions including current release, the larger value will be prioritized if conflicted with `days`',
      min: 1,
      max: 10_000,
    }),
  };

  commandName = 'merge';

  async run(): Promise<void> {
    const {
      flags: { bucket, env, days, latest },
    } = await this.parse(Merge);

    const bucketName = bucket ?? this.getBucketName(env);
    const projectId = this.getProjectId(env);
    const client = await this.getStorageClient(projectId, bucketName);
    try {
      const localDistDir = this.getLocalDistDir();
      const filenames = await this.listAllFiles(client);
      const releases = this.getMergeTargetReleases({ filenames, latest, days });
      const fileEntries = this.getReleasesFileEntries({ filenames, releases });
      await this.downloadRemoteFiles({
        client,
        fileEntries,
        downloadTo: localDistDir,
      });
      // マージされたdistをgcs上の `live` フォルダーにアップロードする
      await client.deleteFiles({ prefix: 'live/' });
      const count = await this.uploadFilesInDir(client, localDistDir, 'live/');
      // ... log
    } catch (error: unknown) {
      // ...
      throw error;
    }
  }

  async downloadRemoteFiles({
    client,
    fileEntries,
    downloadTo,
  }: Readonly<{
    client: Bucket;
    fileEntries: Array<[string, string]>;
    downloadTo: string;
  }>) : Promise<void>;

  getMergeTargetReleases({
    filenames,
    latest,
    days,
  }: Readonly<{
    filenames: readonly string[];
    latest?: number;
    days?: number;
  }>) : string[];

  /*
   * 直近のリリースに存在するユニークなファイルだけを見つけ出し、
   * ファイル名とバケット上のパスを取得する。ファイル名に競合がある
   * 場合はより新しくリリースされたファイルがダウンロード対象となる。
   *
   * @returns Array<[fileName, fileFullPath]>
   */
  getReleasesFileEntries({
    filenames,
    releases,
  }: {
    filenames: readonly string[];
    releases: readonly string[];
  }): Array<[string, string]>;

  filterReleasesByDay(releases: readonly string[], days: number): string[];
}
```
:::

#### `purge`

削除も日数と回数両方対応しています。より広い範囲の方が優先されて、キープ対象（削除しない）になります。

:::details purge
```ts
import { Flags } from '@oclif/core';
import Base from './base';

export default class Purge extends Base {
  static description =
    'multi-release purge command, fetch multiple build version files and purge them into a single build, upload to remote storage bucket';

  static examples = [
    `$ app-cli multi-release purge --bucket=... --days=... --partition=... --latest=2`,
  ];

  static flags = {
    ...Base.flags,
    keepDays: Flags.integer({
      description:
        'delete releases except ones in last n days, the larger value will be prioritized if conflicted with `keepLatest` and lesser releases will be deleted',
      min: 1,
      max: 10_000,
    }),
    keepLatest: Flags.integer({
      description:
        'delete releases except ones n versions, the larger value will be prioritized if conflicted with `keepDays` and lesser releases will be deleted',
      // 最小で直近の２バージョンを保持
      min: 2,
      max: 10_000,
    }),
  };

  commandName = 'purge';

  async run(): Promise<void> {
    const {
      flags: { bucket, env, keepDays, keepLatest },
    } = await this.parse(Purge);

    const bucketName = bucket ?? this.getBucketName(env);
    const projectId = this.getProjectId(env);
    const client = await this.getStorageClient(projectId, bucketName);
    if (keepDays == null && keepLatest == null) {
      throw new Error('Either `keepDays` or `keepLatest` are required');
    }
    try {
      const filenames = await this.listAllFiles(client);
      const releases = this.findReleases(filenames);
      const targetReleases = this.getDeleteTargetReleases({
        releases,
        keepDays,
        keepLatest,
      });

      await Promise.all(targetReleases.map(prefix => client.deleteFiles({ prefix })));
    } catch (error: unknown) {
      // ...
      throw error;
    }
  }

  getDeleteTargetReleases({
    releases,
    keepLatest,
    keepDays,
  }: Readonly<{
    releases: readonly string[];
    keepLatest?: number;
    keepDays?: number;
  }>): string[];

  filterDeleteTargetsByDay(releases: readonly string[], keepDays: number): string[];
}

```
:::

### まとめ

このトライでできたこととして

- 過去のリリースのファイルと最新のリリースのファイルを同時にホストすること
- 自動化のワークフローを利用し、upload -> merge -> purgeの流れで整理できたこと
- oclifのCLIコマンドによって、手動実行が必要な場面にも対応可能になること


## 終わりに

今回のアプリケーションはリリースしてからまもなく1年になります。プロジェクトとしては0->1の段階が終わっていますが、リリースのたびにメンテナンス時間を取らないといけない、との保守運用の課題がずっと残っていました。

バックエンドの方は、時間のかかる処理をCloudRun Jobsに任せることによって、非停止できるようになりましたが、フロントエンドは`Failed to fetch dynamically imported module`の関係で滞っていました。

結構時間をかけて、調査と実験の繰り返しを経て、様々な視点から対策を打つようにしました。この前無事にリリースでき、おかげでその後は早朝深夜のメンテナンス時間なしの、ヘルシーなメンタルに戻りました（笑）。SPAのリリースで、同じ問題で悩んでいる方の役に立てると嬉しく思います。

最後に、今回の問題を立ち向かう際にチームリーダー@lumaさんから多くサポートとレビューをいただきました。特に複数のバージョンのファイルを提供する設計において貴重なアイディアとアドバイスのおかげで今の形で治っています。この場を借りて感謝したいと思います。


