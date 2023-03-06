---
title: "Next.js のフォームでページ遷移前に confirm を表示したい"
emoji: "🔙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "nextjs", "nextrouter"]
published: false
publication_name: leaner_dev
---

# やりたいこと
フォーム入力中にページ遷移をしようとした際、入力データが失わる旨を confirm で警告したい。

# 前提

```
next@13.1.5
@sentry/nextjs@7.38.0
```

# TL;DR
Next.js のフォームでページ遷移前に処理を挟む場合は、`beforeunload` を使った実装だけでは App 内でのページ遷移に対応できない。

そこで、ユーザーの操作方法によって３種類のイベントハンドラの登録が必要になる。

1. App 外ページへの移動 or リロード → `beforeunload`
2. App 内ページへの移動 → `routeChangeStart`
3. App 内ページへのブラウザバック → `beforePopState`

また、`routeChangeStart` での実装で `throw` をする必要があるため、Sentry を使っている場合は通知を無視する設定が必要になる。

<!-- textlint-disable -->
:::details 実装結果
<!-- textlint-enable -->

```tsx:usePageLeaveConfirmation.tsx
import { useRouter } from 'next/router'
import { useState, useEffect } from 'react'

export const usePageLeaveConfirmation = (disabled = false) => {
  const router = useRouter()
  const [isBrowserBack, setIsBrowserBack] = useState(false)

  useEffect(() => {
    const message = 'このページから移動しますか？入力された内容は保存されません。'

    // 1. App外ページへの遷移 or ブラウザリロード
    const beforeUnloadHandler = (event: BeforeUnloadEvent) => {
      event.preventDefault()
      // これがないとChromeなどの一部ブラウザで動作しない
      event.returnValue = ''
    }

    // 2. App内ページへの遷移
    const pageChangeHandler = () => {
        // beforePopStateですでのconfirm表示している場合はスキップ
      if (!isBrowserBack && !window.confirm(message)) {
        throw 'changeRoute aborted'
      }
    }

    // 3. App内ページへのブラウザバック
    const setBeforePopState = () => {
      router.beforePopState(() => {
        if (!confirm(message)) {
          // 書き換わってしまったURLを戻す
          window.history.pushState(null, '', router.asPath)
          return false
        }
        // routeChangeStartで再度confirm表示されるのを防ぐ
        setIsBrowserBack(true) 
        return true
      })
    }
    const clearBeforePopState = () => {
      router.beforePopState(() => true)
    }

    if (!disabled) {
      window.addEventListener('beforeunload', beforeUnloadHandler)
      router.events.on('routeChangeStart', pageChangeHandler)
      setBeforePopState()
      return () => {
        window.removeEventListener('beforeunload', beforeUnloadHandler)
        router.events.off('routeChangeStart', pageChangeHandler)
        clearBeforePopState()
      }
    }
  }, [disabled, isBrowserBack, router])
}
```

```js:sentry.client.config.js
import * as Sentry from '@sentry/nextjs'

Sentry.init({
  ignoreErrors: ['Non-Error promise rejection captured with value: changeRoute aborted'],
})
```

:::

# 実装方針
- 複数のフォームで利用することを想定してカスタムフックとして実装
- 未入力のときなど confirm を表示したくないタイミングを考慮する

# 実装手順
## 0. カスタムフックの準備
大枠の実装は下記の通り。

- 未入力時などは confirm を表示しないようにする。
- コンポーネントのアンマウント時にイベントハンドラの登録解除を忘れずに。

```tsx:usePageLeaveConfirmation.tsx
import { useEffect } from 'react'

export const usePageLeaveConfirmation = (disabled = false) => {
  useEffect(() => {
    // イベントハンドラの定義

    if (!disabled) {
      // イベントハンドラの登録
      return () => {
        // イベントハンドラの登録解除
      }
    }
  }, [disabled])
}
```

呼び出し側のイメージはこんな感じ。
（細かいところはかなり省略している。）

```tsx:SampleForm.tsx
const SampleForm: React.FC<Props> = ({}) => {
  const {
    formState: { isDirty },
  } = useForm<SampleItem>()

  usePageLeaveConfirmation(!isDirty)
}
```

## 1. App 外ページへの移動 or リロード / `beforeunload`
まずは `next/router` に依存しない App 外ページへの移動について考える。

こちらは特になんの変哲もない `beforeunload` へのイベントハンドラ登録。
ただし、[MDN](https://developer.mozilla.org/ja/docs/Web/API/Window/beforeunload_event#%E4%BE%8B) に記載があるように、互換性のため `event.returnValue` も書く必要がある。

```tsx:usePageLeaveConfirmation.tsx
import { useEffect } from 'react'

export const usePageLeaveConfirmation = (disabled = false) => {
  useEffect(() => {
    const message = 'このページから移動しますか？入力された内容は保存されません。'

    // 1. App外ページへの遷移 or ブラウザリロード
    const beforeUnloadHandler = (event: BeforeUnloadEvent) => {
      event.preventDefault()
      // これがないとChromeで動作しない
      event.returnValue = ''
    }

    if (!disabled) {
      window.addEventListener('beforeunload', beforeUnloadHandler)
      return () => {
        window.removeEventListener('beforeunload', beforeUnloadHandler)
      }
    }
  }, [disabled])
}
```

ちなみに `event.returnValue = 'このページから移動しますか？'` のように書いても動くが、メッセージ部分は無視される。

## 2. App 内ページへの移動 / `routeChangeStart`
次に `next/router` に依存する App 内ページへの移動について。

今度は `router.events.on()` を使って `routeChangeStart` にイベントハンドラを登録する必要がある。

```tsx:usePageLeaveConfirmation.tsx
import { useRouter } from 'next/router'
import { useEffect } from 'react'

export const usePageLeaveConfirmation = (disabled = false) => {
  const router = useRouter()

  useEffect(() => {
    // 2. App内ページへの遷移
    const pageChangeHandler = () => {
      const message = 'このページから移動しますか？入力された内容は保存されません。'
      if (!window.confirm(message)) {
        throw 'changeRoute aborted'
      }
    }

    if (!disabled) {
      router.events.on('routeChangeStart', pageChangeHandler)
      return () => {
        router.events.off('routeChangeStart', pageChangeHandler)
      }
    }
  }, [disabled, isBrowserBack, router])
}
```

このとき、ページ遷移をキャンセルするために `throw` を使う必要があるが、このままだと Sentry にエラーが通知されてしまう。

```
UnhandledRejection
Non-Error promise rejection captured with value: changeRoute aborted
```

なので、`sentry.client.config.js` に `ignoreErrors` を追記する必要がある。

```js:sentry.client.config.js
import * as Sentry from '@sentry/nextjs'

Sentry.init({
  ignoreErrors: ['Non-Error promise rejection captured with value: changeRoute aborted'],
  // 正規表現を使っても書ける
  // ignoreErrors: [/changeRoute aborted$/],
})
```


## 3. App 内ページへのブラウザバック / `beforePopState`
ブラウザバックは App 内ページへの移動とは別で考える必要がある。というのも、`routeChangeStart` を使った実装では **ページ遷移をキャンセルしても URL が書き換わってしまう。**

そこで [`beforePopState`](https://nextjs.org/docs/api-reference/next/router#routerbeforepopstate) を使ってブラウザバック前にだけ history を書き換える処理を挟む必要がある。

```tsx:usePageLeaveConfirmation.tsx
import { useRouter } from 'next/router'
import { useEffect } from 'react'

export const usePageLeaveConfirmation = (disabled = false) => {
  const router = useRouter()

  useEffect(() => {
    const message = 'このページから移動しますか？入力された内容は保存されません。'

    // App内ページへのブラウザバック
    const setBeforePopState = () => {
      router.beforePopState(() => {
        if (!confirm(message)) {
          // 書き換わってしまったURLを戻す
          window.history.pushState(null, '', router.asPath)
          return false
        }
        return true
      })
    }
    const clearBeforePopState = () => {
      router.beforePopState(() => true)
    }

    if (!disabled) {
      setBeforePopState()
      return () => {
        clearBeforePopState()
      }
    }
  }, [disabled, router])
}
```

また、`beforePopState` の処理の後に App 内ページ移動のために `routeChangeStart` へ登録した処理が走ってしまう。
それをスキップするために `isBrowserBack` のようなフラグが必要になる。

```tsx:usePageLeaveConfirmation.tsx
import { useRouter } from 'next/router'
import { useState, useEffect } from 'react'

export const usePageLeaveConfirmation = (disabled = false) => {
  const router = useRouter()
  const [isBrowserBack, setIsBrowserBack] = useState(false)

  useEffect(() => {
    const message = 'このページから移動しますか？入力された内容は保存されません。'

    // 2. App内ページへの遷移
    const pageChangeHandler = () => {
      // beforePopStateですでのconfirm表示している場合はスキップ
      if (!isBrowserBack && !window.confirm(message)) {
        throw 'changeRoute aborted'
      }
    }

    // 3. App内ページへのブラウザバック
    const setBeforePopState = () => {
      router.beforePopState(() => {
        if (!confirm(message)) {
          // 書き換わってしまったURLを戻す
          window.history.pushState(null, '', router.asPath)
          return false
        }
        // routeChangeStartで再度confirm表示されるのを防ぐ
        setIsBrowserBack(true) 
        return true
      })
    }
    const clearBeforePopState = () => {
      router.beforePopState(() => true)
    }

    if (!disabled) {
      router.events.on('routeChangeStart', pageChangeHandler)
      setBeforePopState()
      return () => {
        router.events.off('routeChangeStart', pageChangeHandler)
        clearBeforePopState()
      }
    }
  }, [disabled, isBrowserBack, router])
}
```

# まとめのお気持ち
ページから離れる時と言ってもいろいろと考慮することあるのがわかりました。

今回は実装方法について網羅的に解説しているページを見つけられなかったため自分なりに書いてみたが、不具合や考慮漏れの可能性があればコメントをもらえると嬉しいです！
