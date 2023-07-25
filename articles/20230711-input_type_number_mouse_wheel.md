---
title: "input[type=\"number\"] のマウスホイール事故を防ぎたい"
emoji: "🖱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["html", "react", "frontend"]
published: false
publication_name: leaner_dev
---

Leaner Technologies エンジニアのぐりこ( [@glico800](https://twitter.com/glico800) ) です。

今回は何かと話題に上がりがちな `input[type="number"]` について、個人的に新しい発見があったので簡単にまとめてみました！

# 背景
ユーザーから入力した値と保存されている値が違うとのお問い合わせがあり、「そんなはずは…。」と思って調べてみたところ、 `input[type="number"]` の入力後にマウスホイール操作でページ下部にある保存ボタンに移動する際に入力値が意図せず変更されてしまっていたことが発覚。

これは気づけない！ということなんとかしたい。

# やりたいこと
`input[type="number"]` でマウスホイール操作による入力値の変更をできないようにする。

# 前提
- 今回はPC版Chrome `バージョン: 114.0.5735.198` で表示したケースを想定

```
"typescript": "5.1.3",
"react": "18.2.0",
```


# 実装方針
`input[type="number"]` をラップする InputNumber コンポーネントを作成し、その中で `onWheel` イベントによる入力値の変更をできないようにする。

# 実装方法
## 1. InputNumber コンポーネント作る
ここはよくあるラッパーコンポーネントなので詳細は割愛。
（外側から `class` を付与する部分など、何箇所か省略した実装あり）

```tsx:InputNumber.tsx
import React, { forwardRef } from 'react'

export type InputNumberProps = {
  // add your own props
} & React.DetailedHTMLProps<React.InputHTMLAttributes<HTMLInputElement>, HTMLInputElement>

const InputNumber = forwardRef<HTMLInputElement, InputNumberProps>(
  (props, ref) => {
    return (
      <input
        type="number"
        ref={ref}
        {...props}
      />
    )
  }
)
InputNumber.displayName = 'InputNumber'

export default InputNumber
```

## 2. onWheel イベントをキャンセルする

上記にイベントのキャンセル処理を追加したコードがこちら。

```tsx:InputNumber.tsx
import React, { forwardRef } from 'react'

export type InputNumberProps = {
  // add your own props
} & React.DetailedHTMLProps<React.InputHTMLAttributes<HTMLInputElement>, HTMLInputElement>

// イベントハンドラを定義
const onWheelHandler = (e: React.WheelEvent<HTMLInputElement>) => {
  e.currentTarget.blur()
  e.stopPropagation()
}

const InputNumber = forwardRef<HTMLInputElement, InputNumberProps>(
  ({ ...props }, ref) => {
    return (
      <input
        type="number"
        ref={ref}
        // onWheelにイベントハンドラを登録
        onWheel={onWheelHandler}
        {...props}
      />
    )
  }
)
InputNumber.displayName = 'InputNumber'

export default InputNumber
```

追加したイベントハンドラ部分は以下の通り。

```tsx
const onWheelHandler = (e: React.WheelEvent<HTMLInputElement>) => {
  e.currentTarget.blur()
  e.stopPropagation()
}
```

イベントの開始時に `e.currentTarget.blur()` で入力欄からフォーカスを外すことでマウスホイール操作による数値変更を防止。

また、マウスホイール操作の伝播を防止するため、 `e.stopPropagation()` も設定しておく。

# おわり
PC の数値入力むずかしい（小並感）