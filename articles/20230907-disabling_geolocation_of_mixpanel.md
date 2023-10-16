---
title: "Mixpanelで位置情報を表示しないようにする"
emoji: "🌍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mixpanel"]
published: true
publication_name: leaner_dev
---

Leaner Technologies エンジニアのぐりこ( [@glico800](https://twitter.com/glico800) ) です。

今回は最近お試しで導入している Mixpanel の小ネタです！

# やりたいこと
Mixpanel ではデフォルトでイベント発生源の IP アドレスから位置情報を取得・表示するようになっているが、プライバシーの観点から取得しないようにしたい。

![デフォルトの位置情報表示](https://storage.googleapis.com/zenn-user-upload/61a5b9275eb7-20230907.png)
*デフォルトの位置情報表示*

# 前提

```
next@13.4.16
mixpanel-browser@2.47.0
```

# 実現方法
Mixpanel 公式ドキュメントの **Privacy Friendly Tracking** に書かれている [Disabling Geolocation](https://docs.mixpanel.com/docs/tracking/how-tos/privacy-friendly-tracking#disabling-geolocation) で位置情報の取得を無効化できる。

```tsx:pages/_app.tsx
mixpanel.init(yourMixpanelPluginKey, {
  ...,
  ip: false,  // add this option.
})
```

これで以下のように Region や City/Country などのプロパティが `(not set)` になる。

![無効化後の位置情報表示](https://storage.googleapis.com/zenn-user-upload/6bdd0b2cd28f-20230907.png)
*無効化後の位置情報表示*

# おまけ
最初プロパティを非表示にする方法を調べていて、以下の情報にたどり着いた。
ただ、これは使わなくなったプロパティを Filter のプルダウンに表示されるプロパティ一覧に表示したくない場合の話で今回の件とは関係なかった。

[Now you can hide events and properties](https://mixpanel.com/blog/now-you-can-hide-events-and-properties/)

（また、上記のブログ投稿時から UI が変わったのか、もしくは料金プランに依存する機能なのか不明だが、ブログに描かれている手順で設定画面にたどり着けなかった。）
