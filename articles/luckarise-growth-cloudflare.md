---
title: "個人開発アプリの集客を「測れる形」にする — Cloudflare Workers で紹介元つきリンクと Play・AdMob の自動集計"
emoji: "📈"
type: "tech"
topics: ["cloudflare", "cloudflareworkers", "android", "googleplay", "admob"]
published: true
---

Android アプリ [Luckarise](https://luckarise.surc.online/go/zenn?lang=ja&c=zenn-growth) を Google Play に出しました。公開直後の悩みは「どこで紹介すれば入れてもらえるのか」が分からないことです。

そこで、宣伝を始める前に**どの紹介元から何人来たかを毎日自動で数える仕組み**を Cloudflare の無料枠で作りました。この記事はその設計メモです。

## 作ったもの

| 部品 | 役割 |
|---|---|
| 紹介ページ（日英西） | アプリの説明と Google Play へのボタン |
| `/go/<チャネル>` | クリックを数えてから、UTM 付きで Google Play へ転送 |
| 管理画面 `/admin` | 自分だけが見られるダッシュボード（Cloudflare Access） |
| 毎日の Cron | Play のインストール数・AdMob の収益・クリック数を集計 |

すべて 1 つの Worker（Hono）と D1 で動いています。

## 1. 紹介元つきリンク

Google Play は、ストアの URL に `referrer` パラメータを付けると、Play Console のコンバージョン分析で `utm_source` / `utm_campaign` ごとに数字を分けて見せてくれます（[Play Console ヘルプ](https://support.google.com/googleplay/android-developer/answer/9859173)、アプリ側で受け取る場合は [Install Referrer](https://developer.android.com/google/play/installreferrer)）。訪問者が少ない値は「その他」にまとめられる点には注意です。注意点は、`referrer` の値を **丸ごと URL エンコード**することです。

```ts
export function playUrl(o: { channel: string; campaign?: string; lang?: string }) {
	const referrer = `utm_source=${o.channel}&utm_medium=${MEDIUM[o.channel]}&utm_campaign=${o.campaign ?? "none"}`;
	const p = new URLSearchParams({ id: PACKAGE_ID });
	if (o.lang) p.set("hl", o.lang);
	p.set("referrer", referrer); // URLSearchParams が = と & をエンコードしてくれる
	return `https://play.google.com/store/apps/details?${p}`;
}
```

投稿やプロフィールに貼るのは Play の URL ではなく、`https://<自分のドメイン>/go/youtube?lang=ja&c=short-001` のような自前の短いリンクです。Worker はここでクリックを 1 行記録してから 302 で Play に送ります。

- 記録するのは **時刻・チャネル・キャンペーン・言語・国コードだけ**。IP や User-Agent は保存しません。
- リンクプレビューのボットや `HEAD` リクエストは数えません（数字が水増しされるため）。
- 転送先は固定の `play.google.com` だけなので、オープンリダイレクトにはなりません。

## 2. インストール数は API では取れない

最初の驚きはこれでした。**Play Developer Reporting API で取れるのはクラッシュや ANR などの品質指標だけ**で、インストール数は取れません（[リファレンス](https://developers.google.com/play/developer/reporting/reference/rest)）。

代わりに使えるのが、Play が Google Cloud Storage に毎日出力する一括レポートです（[公式](https://support.google.com/googleplay/android-developer/answer/6135870)）。

- 置き場所は `gs://pubsite_prod_<デベロッパーID>/stats/installs/...`
- 月ごとの CSV で、数日遅れで追記される
- **文字コードは UTF-16**（ここでつまずきやすい）

読み取り専用のサービスアカウントを Play Console に招待し、Worker から JWT bearer でトークンを取って GCS の JSON API で読みます。SDK は使わず WebCrypto で RS256 署名しています。

```ts
export function decodeReport(buf: ArrayBuffer): string {
	const b = new Uint8Array(buf);
	if (b[0] === 0xff && b[1] === 0xfe) return new TextDecoder("utf-16le").decode(b.subarray(2));
	return new TextDecoder("utf-8").decode(b);
}
```

招待してから実際に読めるようになるまでは時間がかかり、その間は 403 が返ります。毎日の Cron は失敗をそのまま記録し、ほかの集計は止めない作りにしました。

## 3. AdMob はサービスアカウントが使えない

AdMob API はサービスアカウントに対応していないので、本人の OAuth（refresh token）を使います。

- OAuth アプリを「テスト中」のままにすると、**refresh token が 7 日で失効**します。本番に切り替えておきます。
- 本番に切り替えるには、ホームページ・プライバシーポリシー・承認済みドメインがそろっている必要がありました。プライバシーポリシーが別ドメインだったので、Worker に `/privacy` → 本体への 302 を置いて同じドメインにそろえました。
- refresh token は手元のスクリプト（PKCE＋ループバック）で受け取り、そのまま `wrangler secret put` に標準入力で渡します。画面にもログにも出しません。

## 4. 管理画面は Cloudflare Access ＋ Worker 側の JWT 検証

`/admin` は Cloudflare Access（Zero Trust の無料枠）で自分のメールだけに絞り、さらに Worker 側でも `Cf-Access-Jwt-Assertion` を検証しています。

- RS256 固定・`aud`・`exp`・`nbf`・`iss`・メールの許可リストを確認
- 設定値が無いときは **常に 403**（fail closed）
- `workers_dev = false` にして、カスタムドメイン以外から `/admin` に届かないようにする
- POST は `Origin` が自分と一致するものだけ受け付ける

## 5. 「効いた施策だけを残す」ための実験ログ

ダッシュボードには、チャネルごとの 7 日・28 日のクリック数、キャンペーン別の表、Play のインストール数、AdMob の収益に加えて、**実験ログ**を置きました。施策を始めたら「やること・仮説・開始日」を 1 行書き、2 週間以上たってから数字で「効果あり／なし」を閉じます。効果があったものだけを手順として残していく運用です。

## まとめ

- 宣伝の前に、**紹介元ごとに数えられるリンク**を用意する
- インストール数は API ではなく **GCS の一括レポート（UTF-16）** から取る
- AdMob は **OAuth を本番**にしてから refresh token を取る
- 管理画面は **Access と Worker 側の JWT 検証の二重**にする

この記事のリンクも `/go/zenn` 経由にしてあるので、この記事から何人がアプリを見に来たかも数えています。

アプリはこちら: [Luckarise（Google Play）](https://luckarise.surc.online/go/zenn?lang=ja&c=zenn-growth)
