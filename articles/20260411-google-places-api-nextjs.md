---
title: "Google Places API (New) をNext.jsに繋ぐまでの地味な手順"
emoji: "🗺️"
type: "tech"
topics: ["googlecloud", "nextjs", "typescript", "googlemaps", "api"]
published: true
---

## 今日やったこと

Next.js で作っている飲食店マップアプリで、地図ライブラリを react-leaflet から
Google Maps に移行した。それに伴って Google Places API (New) も導入したのだが、
「動くまでの手順」が意外と細かくて、整理しておきたくなった。

## GCP 側でやること

まず Google Cloud Console での作業から。ここを雑にやると後で詰まる。

**1. プロジェクト作成**
既存プロジェクトがあれば流用でもいいが、API 管理を分けたいなら新規作成が無難。

**2. 有効化する API は2種類**
- Maps JavaScript API（ブラウザ側のマップ表示）
- Places API (New)（サーバー側の店舗検索）

"Places API" と "Places API (New)" は別物なので注意。
`POST https://places.googleapis.com/v1/places:searchNearby` を使うなら
必ず **(New)** の方を有効化する。

**3. APIキーの制限設定**
- Maps JavaScript API 用 → HTTP リファラー制限（`localhost:3000/*` など）
- Places API (New) 用 → IP アドレス制限またはサーバー限定

ブラウザ公開鍵をサーバー側でも使い回すのは避けたほうがいい。
用途が違うのでキーを分けておく。

**4. 請求アラートの設定**
Maps 系 API は従量課金なので、Cloud Billing でアラートを必ず設定する。
月 $200 の無料枠があるが、バグでループリクエストが走ると一瞬で溶ける。

## Next.js 側でやること

```bash
npm install @googlemaps/js-api-loader
```

v2 系では `Loader` クラスではなく、関数ベースの API を使う。

```typescript
import { setOptions, importLibrary } from "@googlemaps/js-api-loader";

setOptions({
  key: process.env.NEXT_PUBLIC_GOOGLE_MAPS_API_KEY ?? "",
  v: "weekly",
  libraries: ["places", "marker"],
});

// コンポーネント内で
const { Map } = await importLibrary("maps") as google.maps.MapsLibrary;
const { AdvancedMarkerElement } = await importLibrary("marker") as google.maps.MarkerLibrary;
```

v1 の `new Loader({ apiKey, version, libraries })` から書き方が変わっているので、
古い記事を参考にしていると型エラーに気づかず詰まる（詰まった）。

**.env の整理**

```
NEXT_PUBLIC_GOOGLE_MAPS_API_KEY=""   # ブラウザ側
GOOGLE_MAPS_API_KEY=""               # サーバー側
```

`NEXT_PUBLIC_` プレフィックスがあるとクライアントバンドルに含まれる。
サーバー専用キーは絶対に `NEXT_PUBLIC_` を付けない。

## ハマりポイントまとめ

- **Places API と Places API (New) は別物**。Nearby Search の新エンドポイントを使うなら (New)
- **`@googlemaps/js-api-loader` v2 は関数 API**。`Loader` クラスは非推奨
- **`apiKey` → `key`、`version` → `v`** にパラメータ名が変わっている
- **AdvancedMarkerElement に mapId が必要**。Cloud Console で Map ID を作成して
  `NEXT_PUBLIC_GOOGLE_MAPS_API_KEY` のキーと紐付けておく

## まとめ

GCP 側の API 有効化・キー分離・請求アラートを先に済ませてから実装に入るのが正解だった。
「とりあえず動かしてから設定する」でいくと、後から制限の掛け直しが面倒になる。
Places API (New) のレスポンス形式は旧 API と結構違うので、型定義を先に見ておくといい。
