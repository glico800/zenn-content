---
title: "JavaScriptで配列を破壊しない方法まとめ"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript", "typescript", "配列"]
published: true
publication_name: leaner_dev
---

リーナー購買開発チームの [@glico800](https://x.com/glico800) です。

最近鹿野さんのポストを見て、ECMAScript2023 で追加された配列の非破壊メソッドのことを知らなかったので、新メソッド追加後の「配列を破壊しない方法」をまとめてみました。

https://x.com/tonkotsuboy_com/status/1974076338027016274

# はじめに
## 配列操作における破壊・非破壊とは？
※普段、破壊・非破壊という表現を使わない言語を触っている方向けです。


配列などのデータ構造に対する処理のうち、元のデータ構造の値を直接書き換えるものを「破壊的」、元のデータ構造の値を維持するものを「非破壊的」と呼びます。メソッドの場合もその性質に応じて「破壊的メソッド」「非破壊的メソッド」のように呼びます。

ちなみに破壊的操作は `in-place` という用語を使って説明されることが多いので、こちらの方がピンと来る人も多いのではないでしょうか？

## 破壊的メソッド＝絶対悪ではない
「破壊的メソッドを絶対に使うな！」という趣旨の記事ではありません。

[鹿野さんのポストへの引用ポスト](https://x.com/kensukesaitocom/status/1974244883155083584)でも言及されている方がいましたが、既に一時配列になっている状態で非破壊メソッドを使うと、必要以上にメモリを消費するなどのデメリットもあるので、あくまで適材適所です。

## 記事内サンプルコードの実行環境について
一部 TypeScript のサンプルコードがあるため、実際に手元で動かしてみたい場合は [TS Playground](https://www.typescriptlang.org/play/) での実行がおすすめです。

ただし、`toSorted()` `toReversed()` `toSpliced()` `with()` は ECMAScript2023 / Node.js 20 以降でないと使えません。[TS Playground](https://www.typescriptlang.org/play/) を使う場合は TS Config から「Target: ES2023」に設定してから実行してください。


# 本編
## 1. `sort()` → `toSorted()`
配列の要素をソートしたい場合、破壊的メソッドである [`sort()`](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/sort) の代わりに非破壊版の [`toSorted()`](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/toSorted) が使えます。


```js:破壊的
const numbers = [2, 3, 1, 4]

numbers.sort()
console.log(`numbers (after sort): ${numbers}`) // => 1,2,3,4
```

```js:非破壊的
const numbers = [2, 3, 1, 4]

const sortedNumbers = numbers.toSorted()
console.log(`numbers (after toSorted): ${numbers}`) // => 2,3,1,4
console.log(`sortedNumbers: ${sortedNumbers}`) // => 1,2,3,4
```

## 2. `reverse()` → `toReversed()`
配列の要素を反転させたい場合、破壊的メソッドである [`reverse()`](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/reverse) の代わりに非破壊版の [`toReversed()`](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/toReversed) が使えます。

```js:破壊的
const numbers = [1, 1, 2, 3, 5, 8]

numbers.reverse()
console.log(`numbers after reverse: ${numbers}`) // => 8, 5, 3, 2, 1, 1
```

```js:非破壊的
const numbers = [1, 1, 2, 3, 5, 8]

const reversedNumbers = numbers.toReversed()
console.log(`numbers after toReversed: ${numbers}`) // => 1, 1, 2, 3, 5, 8
console.log(`reversedNumbers: ${reversedNumbers}`) // => 8, 5, 3, 2, 1, 1
```

## 3. `splice()` → `toSpliced()`
指定された位置の要素を削除・置換したり、新しい要素を追加したりしたい場合、破壊的メソッドである [`splice()`](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/splice) の代わりに非破壊版の [`toSpliced()`](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/toSpliced) が使えます。

### 3-1.要素の削除
```js:破壊的
const characters = ['Taro', 'Hanako', 'Wally', 'Jiro']
const whereIsWally = characters.indexOf('Wally')

characters.splice(whereIsWally, 1)
console.log(`characters after splice: ${characters}`) // => Taro,Hanako,Jiro
```

```js:非破壊的
const characters = ['Taro', 'Hanako', 'Wally', 'Jiro']
const whereIsWally = characters.indexOf('Wally')

const others = characters.toSpliced(whereIsWally, 1)
console.log(`characters after toSpliced: ${characters}`) // => Taro,Hanako,Wally,Jiro
console.log(`others: ${others}`) // => Taro,Hanako,Jiro
```

### 3-2.要素の置換
```js:破壊的
const characters = ['Taro', 'Hanako', 'Wally', 'Jiro']
const whereIsWally = characters.indexOf('Wally')

characters.splice(whereIsWally, 1, 'XXXX')
console.log(`characters after splice: ${characters}`) // => Taro,Hanako,XXXX,Jiro
```

```js:非破壊的
const characters = ['Taro', 'Hanako', 'Wally', 'Jiro']
const whereIsWally = characters.indexOf('Wally')

const maskedCharacters = characters.toSpliced(whereIsWally, 1, 'XXXX')
console.log(`characters after toSpliced: ${characters}`) // => Taro,Hanako,Wally,Jiro
console.log(`maskedCharacters: ${maskedCharacters}`) // => Taro,Hanako,XXXX,Jiro
```

### 3-3.要素の追加
```js:破壊的
const characters = ['Taro', 'Hanako', 'Wally', 'Jiro']
const whereIsWally = characters.indexOf('Wally')

characters.splice(whereIsWally, 0, 'bunsin1')
characters.splice(whereIsWally + 2, 0, 'bunsin2')
console.log(`characters after splice: ${characters}`) // => Taro,Hanako,bunsin1,Wally,bunsin2,Jiro
```

```js:非破壊的
const characters = ['Taro', 'Hanako', 'Wally', 'Jiro']
const whereIsWally = characters.indexOf('Wally')

const bunsin = characters.toSpliced(whereIsWally, 0, 'bunsin1').toSpliced(whereIsWally + 2, 0, 'bunsin2')
console.log(`characters after toSpliced: ${characters}`) // => Taro,Hanako,Wally,Jiro
console.log(`bunsin: ${bunsin}`) // => Taro,Hanako,bunsin1,Wally,bunsin2,Jiro
```

## 4. `splice()` や `array[i] = value` による置換 → `with()`
指定された位置の要素の置換をしたい場合、破壊的操作をする `splice()` や `array[i]=` の代わりに [`with()`](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/with) が使えます。

```js:破壊的
const characters = ['Taro', 'Hanako', 'Wally', 'Jiro']
const whereIsWally = characters.indexOf('Wally')

characters.splice(whereIsWally, 1, '----') // or characters[whereIsWally] = '----'
console.log(`characters after splice: ${characters}`) // => Taro,Hanako,----,Jiro
```

```js:非破壊的
const characters = ['Taro', 'Hanako', 'Wally', 'Jiro']
const whereIsWally = characters.indexOf('Wally')

const maskedCharacters = characters.with(whereIsWally, '----')
console.log(`characters after with: ${characters}`) // => Taro,Hanako,Wally,Jiro
console.log(`maskedCharacters: ${maskedCharacters}`) // => Taro,Hanako,----,Jiro
```

## 5. `unshift()` → `toSpliced()`
配列の先頭に要素を追加したい場合、破壊的メソッドである [`unshift()`](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/unshift) の代わりに前述の [`toSpliced()`](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/toSpliced) が使えます。

```js:破壊的
const titles = ['Z', 'A']

titles.unshift('X', 'Y')
console.log(`titles after unshift: ${titles}`) // => X,Y,Z,A
```

```js:非破壊的
const titles = ['Z', 'A']

const newTitles = titles.toSpliced(0, 0, 'X', 'Y')
console.log(`titles after toSpliced: ${titles}`) // => Z,A
console.log(`newTitles: ${newTitles}`) // => X,Y,Z,A
```

## 6. `push()` → `concat()`
（ES2023 前後で変更無し）
配列の末尾に要素を追加したい場合は、破壊的メソッドである [`push()`](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/push) の代わりに [`concat()`](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/concat) が使えます。

```js:破壊的
const colors = ['red', 'green']

colors.push('blue', 'Pikachu')
console.log(`colors after push: ${colors}`) // => red,green,blue,Pikachu
```

```js:非破壊的
const colors = ['red', 'green']

const newColors = colors.concat(['blue', 'Pikachu'])
console.log(`colors after concat: ${colors}`) // => red,green
console.log(`newColors: ${newColors}`) // => red,green,blue,Pikachu
```

## 7. `shift()` `pop()` → `slice()`
（ES2023 前後で変更無し）
配列の先頭・末尾の要素を削除したい場合は、破壊的メソッドである [`pop()`](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/pop) [`shift()`](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/shift) の代わりに [`slice()`](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/slice) が使えます。

```js:破壊的
const planaria = ['head', 'body', 'tail']

planaria.shift()
planaria.pop()
console.log(`planaria after shift&pop: ${planaria}`) // => body
```

```js:非破壊的
const planaria = ['head', 'body', 'tail']

const newPlanaria = planaria.slice(1).slice(0, -1)
console.log(`planaria after slice: ${planaria}`) // => head,body,tail
console.log(`newPlanaria: ${newPlanaria}`) // => body
```

# 補足
## ECMAScript2023 のメソッドが使えない環境ではどうしたら？
[スプレッド構文 (...)](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Spread_syntax) を使って配列を複製してから破壊的操作をしましょう。

```js
const dna = ['A', 'T', 'G', 'C']
const newDna = [...dna]

// newDna に対して破壊的操作をしても dna には影響なし
newDna.pop()
console.log(`dna: ${dna}`) // => A,T,G,C
```

さらにスプレッド構文も使えない！という場合は `concat()` でも配列の複製ができます。

```js
const dna = ['A', 'T', 'G', 'C']
const newDna = dna.concat()
```

## 破壊的メソッドを使うと困るのはどんなとき？
以下の例のように引数の配列に対して破壊的操作をしてしまったときです。

この場合、メソッドの呼び出し元で破壊的操作がされたことに気づかず、引数に渡した配列をさらに別の処理でも使うことで予期せぬエラーに繋がります。

```js
const blackBoxSort = (array: number[]): number[] => {
    return array.sort()
}

const numbers = [2, 3, 1, 4]
const sortedNumbers = blackBoxSort(numbers)

// 引数で渡したnumbersが実は変更されてしまっている
console.log(`numbers: ${numbers}`) // => 1,2,3,4
```

## 引数で Array を受け取るには `readonly` をつけると安心
TypeScript を使っている場合、引数の配列の型を `readonly` にしておくことで、実行前に破壊的操作を検知できます。

```js
const blackBoxSort = (array: readonly number[]): number[] => {
    // Error: Property 'sort' does not exist on type 'readonly number[]'.
    return array.sort()
}
```

# 参考
https://typescriptbook.jp/reference/values-types-variables/array/array-operations
https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array

# 宣伝
リーナーでは一緒に働いてくれるエンジニアを募集しています！
https://careers.leaner.co.jp/engineering
