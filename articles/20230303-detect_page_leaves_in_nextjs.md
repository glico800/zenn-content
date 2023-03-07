---
title: "Next.js のフォームでページ遷移前に confirm を表示したい"
emoji: "🔙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "nextjs", "nextrouter"]
published: false
publication_name: leaner_dev
---

Leaner Technologies エンジニアのぐりこ( [@glico800](https://twitter.com/glico800) ) です。

よくある **「このページから移動しますか？」** みたいな confirm 表示を挟む実装を Next.js を使ったプロダクトでやってみたらいろいろと学びがあったので、備忘録的にまとめてみました。

# やりたいこと
フォーム入力中にページ遷移をしようとした際、入力データが失われる旨を confirm で警告したい。

![表示イメージ](https://storage.googleapis.com/zenn-user-upload/7d0c41f51813-20230307.png)
*こんなの*

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
      // beforePopStateで既にconfirm表示している場合はスキップ
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
- ページ遷移を伴う各操作時に confirm を表示
- 複数のフォームで利用することを想定してカスタムフックとして実装
- 未入力時など confirm を表示しないタイミングを指定できるようにする

# 実装手順
## 0. カスタムフックの準備
大枠の実装は下記の通り。

- `disabled` の値によって未入力時などは confirm を表示しないようにする
- コンポーネントのアンマウント時にイベントハンドラの登録解除を忘れずに

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

```tsx:SampleForm.tsx
import { usePageLeaveConfirmation } from 'hooks/usePageLeaveConfirmation'

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

```tsx:usePageLeaveConfirmation.tsx
import { useEffect } from 'react'

export const usePageLeaveConfirmation = (disabled = false) => {
  useEffect(() => {
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

### 注意点

- [MDN](https://developer.mozilla.org/ja/docs/Web/API/Window/beforeunload_event#%E4%BE%8B) に記載があるように、互換性のため `event.returnValue` も書く必要がある
- 表示するメッセージは指定できない
    - `event.returnValue = 'このページから移動しますか？'` のように書いても動くが、メッセージ部分は無視されてデフォルトのメッセージが表示される

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

### 注意点
ページ遷移をキャンセルするために `throw` を使う必要があるが、このままだと Sentry にエラーが通知されてしまう。

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
    // App内ページへのブラウザバック
    const setBeforePopState = () => {
      router.beforePopState(() => {
        const message = 'このページから移動しますか？入力された内容は保存されません。'

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

### 注意点
`beforePopState` の処理の後に App 内ページ移動のために `routeChangeStart` へ登録した処理が走ってしまう。
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
      // beforePopStateで既にconfirm表示している場合はスキップ
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
ページから離れる時と言ってもいろいろと考慮することがあるのだとわかりました。

今回は自分なりにいろいろと調べて書いてみたが、不具合や考慮漏れの可能性があればコメントをもらえると嬉しいです！
