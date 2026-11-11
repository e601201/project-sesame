# project-sesame ローカル起動手順

Android サンプルアプリ Shrine（`identity-samples/Shrine`）の接続先として、project-sesame をローカルで起動する手順です。2026-11-09 と 2026-11-11 に動作を確認しました。

アプリ側の手順は `identity-samples/Shrine/LOCAL_SETUP.md` を参照してください。

## 必要なもの

| ツール | 用途 | 入手方法 |
|---|---|---|
| Node.js | サーバーとフロントエンド | Homebrew など |
| JDK 21 以上 | Firestore エミュレータ（firebase-tools） | `brew install openjdk@21` |
| Caddy | `https://rp.localhost` のリバースプロキシ（任意） | `brew install caddy` |
| cloudflared | Shrine（端末やエミュレータ）からつなぐためのトンネル | `brew install cloudflared` |

## 1. 初回セットアップ

`npm ci` は、postinstall で Caddy を GitHub からダウンロードします。このダウンロードが失敗する（HTTP 504）ことがあるので、そのときは次の手順で回避します。

```sh
cd ~/Workspace/project-sesame
npm ci --ignore-scripts
npm rebuild esbuild re2 @parcel/watcher protobufjs @firebase/util
cp "$(brew --prefix)/bin/caddy" node_modules/@radically-straightforward/caddy/caddy
ln -s ../@radically-straightforward/caddy/caddy node_modules/.bin/caddy
```

ダウンロードが成功する環境なら、`npm ci` だけで済みます。

## 2. サーバーを起動する

`package.json` の `dev:local` から Caddy を外したものです。Caddy は sudo が必要なので、別に起動します（手順 3）。

```sh
cd ~/Workspace/project-sesame
PATH=/opt/homebrew/opt/openjdk@21/bin:$PATH \
ALLOWED_DEV_HOSTS=.trycloudflare.com \
npx concurrently -k -n "EMU,CLIENT,RP,IDP" \
  "npm:emulator" \
  "wait-on tcp:4000 && npm run dev:client" \
  "wait-on tcp:4000 && NODE_ENV=rp-localhost npm run dev:server" \
  "wait-on tcp:4000 && NODE_ENV=idp-localhost npm run dev:server"
```

| 名前 | 内容 | ポート |
|---|---|---|
| EMU | Firestore エミュレータ。データは `./.data` から読み込み、終了時に書き出す | 8081（Emulator UI は 4000） |
| CLIENT | フロントエンドのビルドと配信（rsbuild） | 3000 |
| RP | RP サーバー（`rp-localhost.config.json`） | 8080 |
| IDP | IdP サーバー（`idp-localhost.config.json`） | 8000 |

- `PATH` に JDK 21 を加えるのは、Firestore エミュレータが JDK 21 以上を必要とするためです。
- `ALLOWED_DEV_HOSTS=.trycloudflare.com` を付けると、トンネル経由のリクエストではそのホスト名が RP ID になります。Shrine からつなぐときに必要です。
- `rp-localhost.config.json` の assetlinks には、Shrine の debug.keystore の SHA256 フィンガープリントが登録済みです。

## 3. Caddy を起動する（任意）

ブラウザから `https://rp.localhost` と `https://idp.localhost` で開きたいときだけ使います。Shrine をつなぐだけなら不要です。sudo が必要なので、別のターミナルで実行します。

```sh
cd ~/Workspace/project-sesame
./caddy.sh
```

## 4. Shrine 用のトンネルを開く

```sh
cloudflared tunnel --no-autoupdate --url http://localhost:8080
```

ログに表示される `https://xxxx.trycloudflare.com` が接続先です。ブラウザか curl で `https://xxxx.trycloudflare.com/.well-known/assetlinks.json` が 200 を返せば、つながっています。

- URL は起動のたびに変わります。そのたびに Shrine をビルドし直してください。
- この URL を知っていれば、誰でもローカルのサーバーにアクセスできます。使い終わったら止めてください。

## 5. 止める

どれも、起動したターミナルで `Ctrl+C` を押すと止まります。Firestore エミュレータのデータは、終了時に `./.data` に保存されます。

## データを確認する

Firestore エミュレータの中身は、Emulator UI（http://localhost:4000）で見られます。REST API でも確認できます。

```sh
curl -s -H 'Authorization: Bearer owner' \
  'http://localhost:8081/v1/projects/project-sesame-426206/databases/(default)/documents/public_key_credentials'
```

`aaguid` が `restore-credential` のドキュメントは、Restore Credentials のリストアキーです。
