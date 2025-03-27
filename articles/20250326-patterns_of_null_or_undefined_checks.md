---
title: "JavaScriptのnull/undefined判定の仕方いろいろ"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript"]
published: false
publication_name: leaner_dev
---

# なんの記事？
JavaScript で `null`/`undefined` 判定の書き方がいろいろあってコードレビューとか説明するとき用に改めてまとめてみた。
どちらかというと普段 JavaScript をあまり書かない人が読むことを想定して書く予定。

# 前提知識
JavaScript では Boolean に変換したときに `true` / `false` になる値のことをそれぞれ [Truthy (真値)](https://developer.mozilla.org/ja/docs/Glossary/Truthy)/[Falsy (偽値)](https://developer.mozilla.org/ja/docs/Glossary/Falsy) と呼ぶ。

# Truthy/Falsyを利用する時の罠
`null`/`undefined` 判定のとき以下のように書きたくなるが、特定のケースで意図した通りに動かなかったり、それを考慮したコードレビューが大変だったりする。

```js
let unitPrice
const formatPrice = (num) => { console.log('run formatPrice') }

// unitPrice が null/undefined のときは formatPrice() を実行しないようにしたい
unitPrice = 100
if (unitPrice) { formatPrice(unitPrice) }
// => 意図通り formatPrice() は実行される

unitPrice = 0
if (unitPrice) { formatPrice(unitPrice) }
// => 0 が Falsy のため実行されない（null/undefined ではないで実行してほしかった）
```

なので、本記事では `if (value)` を使わないパターンのみまとめた。
※[論理否定演算子 (!)](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Logical_NOT) を使う `if (!value)` についても同じく除外

# 先におすすめの書き方
## if文
```js
// null/undefined なら実行する
if(unitPrice == null) { console.log('unit price is null/undefined') }

// null/undefined 以外なら実行する
if(unitPrice != null) { formatPrice(unitPrice) }
```

## 代入
```js
// null/undefined ならデフォルト値（0）を代入
const price = unitPrice ?? 0
```

# 本題：`null`/`undefined` 判定の仕方いろいろ
## `value === null || value === undefined`
[厳密等価演算子 (===)](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Strict_equality)を使って書くパターン。

```js
// null/undefined なら実行する
if (value === null || value === undefined) { console.log('unit price is null/undefined') }

// null/undefined 以外なら実行する
if (value !== null && value !== undefined) { formatPrice(unitPrice) }
```

シンプルで読みやすいが、ちょっと長いのが難点。
長さについては好みの部分もあるので、明示的に書くことを重視するならベターかも。

## `value == null`
[厳密等価演算子 (===)](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Strict_equality)ではなく[等価演算子 (==)](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Equality)を使って書くパターン。

```js
// null/undefined なら実行する
if(unitPrice == null) { console.log('unit price is null/undefined') }

// null/undefined 以外なら実行する
if(unitPrice != null) { formatPrice(unitPrice) }
```

[等価演算子 (==)](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Equality)は比較する値同士の型が違う場合は型変換をしてみてから比較するという仕様のため、`'0'` と `0` のように型が違う値を等価だと判定してしまって困るなどの罠があり、基本的には使わない方がよい。
ただし、前述の利用して `== null` のように書くときは例外的に利用することがある。

なぜ `== null` で `null`/`undefined` を両方判定できるかの説明はややこしいので省くが、簡潔に言うと `null` と `undefined` は型が異なるが値としては同じなので、値だけをみる[等価演算子 (==)](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Equality)では `null`/`undefined` 両方の判定ができる。（なので、`== undefined` と書いても同じ）
この辺りはあまり詳しく書いている公式ドキュメントが見つからなかったが、[null と undefined の違い](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/null#null_%E3%81%A8_undefined_%E3%81%AE%E9%81%95%E3%81%84)あたりを読んでもらうと分かるかもしれない。

どういう仕組みか知っていないと正しく読めないので、このパターンは使わないというルールにしている場合もありそう。

## `value ?? 'default value'`
[Null 合体演算子 (??)](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing)を使って書くパターン。

```js
// null/undefined ならデフォルト値（0）を代入
const price = unitPrice ?? 0
```
[Null 合体演算子 (??)](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing)は左辺が `null`/`undefined` の場合に右辺の値を返し、それ以外の場合に左辺の値を返します。そのため、右辺にデフォルト値を書くことで `null`/`undefined` の場合のみデフォルト値を返すときに利用できる。

### 注意1：[論理和演算子 (||)](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Logical_OR)を使ったパターンとの違い
このパターンは一見すると[論理和演算子 (||)](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Logical_OR)を使った書き方に似ているが、前述の通り`null`/`undefined` かどうかの判定のみをしてくれるので、Truthy/Falsy の罠にかからない。

```js
// null/undefined ならデフォルト値（0）を代入
const price = unitPrice || 0
// => unitPrice が Truthy の場合も unitPrice が代入されてします（例：unitPrice = '0'）
```

↑ のパターンは TypeScript を使っていて `unitPrice` が Number 型であることが分かっているケースなどでは基本的に問題にならないが、以下のようなケースもあるため、個人的には[Null 合体演算子 (??)](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing)を使った書き方に統一するのが望ましいと思う。

```js
// null/undefined ならデフォルト値（空文字）を代入
const price = unitPrice || ''
// => unitPrice が 0 のときに price が空文字になってしまう
```

補足：`typescript-eslint` に [prefer-nullish-coalescing](https://typescript-eslint.io/rules/prefer-nullish-coalescing/) というルールがある。

### 注意2：if 文の中で使うと Truthy/Falsy の罠にかかる可能性あり

```js
// Bad: null/undefined なら実行する
if (unitPrice ?? true) { console.log('unit price is null/undefined') }
// => unitPrice が Truthy の場合も実行されてしまう（例：unitPrice = 100）

// Bad: null/undefined 以外なら実行する
if (unitPrice ?? false) { formatPrice(unitPrice) }
// => unitPrice が Falsy の場合に実行されない（例：unitPrice = 0）
```

# おわり
ややこしい仕様も乗り越えて、よき JavaScript ライフを！