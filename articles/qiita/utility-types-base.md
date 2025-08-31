---
title: 【TypeScript】Utility Typesの型定義元を見て理解を深める
tags: TypeScript JavaScript
author: kikikikimorimori
slide: false
---
### はじめに
TypeScriptには組み込み型関数である`Utility Types`があります。

その型定義元の中身がどうなっているかを見ていこうと思います。

### Partial

```TS
/**
 * Tの全てのプロパティをオプションにする
 */
type Partial<T> = {
    [P in keyof T]?: T[P];
};
```
引数で渡された型`T`を`Mapped Types`を使って、分解し再定義しています。

具体的には、`T`のプロパティを`in`と`keyof`を使って、順番に`P`に代入し、`?`(オプショナル)をつけてオプションにしています。
そして、その値の型を`T[P]`で取り出し再定義しています。

### Required

```TS
/**
 * Tの全プロパティを必須にする
 */
type Required<T> = {
    [P in keyof T]-?: T[P];
};
```

こちらも先ほどの`Partial`と同様に`Mapped Types`を使って、分解し再定義しています。

違う点として、`-?`をつけることで`?`(オプショナル)を外し、プロパティを必須としています。

### Readonly

```TS
/**
 * Tの全てのプロパティを読み取り専用にする
 */
type Readonly<T> = {
    readonly [P in keyof T]: T[P];
};
```
こちらも`Mapped Types`を使って、分解し再定義しています。

再定義する際に、`readonly`をつけることでプロパティを読み取り専用としています。

### Pick

```ts
/**
 * Tから、Kで指定されたプロパティのみにする
 */
type Pick<T, K extends keyof T> = {
    [P in K]: T[P];
};
```

`Pick`は引数を2つとり、`extends`を使うことで、第二引数には、第一引数で指定した型のプロパティのユニオン型のみに制限しています。

そして`Mapped Types`を使い、`K`で指定されたプロパティのみを持つ型に再定義しています。

### Omit
```ts
/**
 * Tから、Kで指定されたプロパティ以外にする
 */

type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;

// ↓↓↓ 展開するとこんな感じになる

type Omit<T, K extends keyof any> = {
  [P in keyof T as P extends K ? never: P]: T[P];
};
```
`keyof T as P`で第一引数に指定された`T`のプロパティを順番に取り出し`P`として定義し、`extends`を使い、そのプロパティが第二引数の型`K`に割り当て可能な場合は`never`、割り当て可能ではない場合は`P`を返します。

そして`Mapped Types`を使い、最終的には`K`で指定されたプロパティ以外を持つ型に再定義しています。

### Record

```TS
/**
 * Kをプロパティ、Tを値の型として持つ型にする
 */
type Record<K extends keyof any, T> = {
    [P in K]: T;
};
```
第一引数を`extends keyof`でユニオン型に制限し、`Mapped Types`を使って、分解し`T`を値の型として持つ型に再定義しています。


### Exclude
```TS
/**
 * Uに割り当て可能な型をTから除外する。
 */
type Exclude<T, U> = T extends U ? never : T;
```
`T`が`U`に割り当て可能な型の場合、`never`、割り当て可能ではない場合、`T`を返します。

ユニオン型から特定のプロパティを除きたい場合に使うことが多いです。

```ts
type Union = Exclude<string | number | boolean, boolean>
// => string | number
```

第一引数にユニオン型を指定した場合、`T extends U ? never : T`の`T`に順番にプロパティが入り、プロパティ分繰り返されます。

```ts
string extends boolean ? never : string
//=> string

number extends boolean ? never : number
//=> number

boolean extends boolean ? never : boolean
//=> never

// ↓↓↓

string | number

```

### Extract

```TS
/**
 * Uに割り当て可能な型をTから抽出する。
 */
type Extract<T, U> = T extends U ? T : never;
```

こちらは先程の`Exclude`の逆で、`T`が`U`に割り当て可能な型の場合、`T`、割り当て可能ではない場合、`never`を返します。

### NonNullable

```TS
/**
 * Tからnullとundefinedを除外する
 */
type NonNullable<T> = T & {};
```
引数`T`と`{}`に対してインターセクション型を使うことで`null`と`undefined`を除外しています。

### Parameters

```TS
/**
 * 関数型のパラメータをタプル型にする
 */
type Parameters<T extends (...args: any) => any> = T extends (...args: infer P) => any ? P : never;
```

引数の型`T`を`extends (...args: any) => any`で関数型に制限します。

`T extends (...args: infer P) => any ? P : never`で`T`が関数型の場合、`infer`で引数（args）のタプル型`P`を取り出し、そのまま`P`を返します。


`infer`は`conditional types`(extends)で条件分岐した際に推論される型情報を入れられる型変数です。

また下記のように、同じ型変数に対して複数の推論場所を持つこともできます。

```ts
type Unpacked<T> = T extends (infer U)[]
  ? U
  : T extends (...args: any[]) => infer U
  ? U
  : T extends Promise<infer U>
  ? U
  : T;
```

これは配列、関数、Promiseの中身の方を取り出す型です。

```ts
type ArrayArg = type Unpacked<string[]>
// => string

type FunctionArg = type Unpacked<(...args: any) => number>
// => number

type PromiseArg = type Unpacked<Promise<boolean>>
// => boolean

```

### ReturnType
```TS
/**
 * 関数の戻り値の型を取得する
 */
type ReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : any;
```

引数の型`T`を`extends (...args: any) => any`で関数型に制限します。

`T extends (...args: any) => infer R ? R : any`で`T`が関数型の場合、`infer`で戻り値の型`P`を取り出し、その`P`を返します。

### ConstructorParameters

```TS
/**
 * コンストラクタ関数型のパラメータをタプルで取得する
 */
type ConstructorParameters<T extends abstract new (...args: any) => any> = T extends abstract new (...args: infer P) => any ? P : never;
```

引数`T`を`extends abstract new (...args: any) => any`でクラスもしくは抽象クラスに制限します。

`T`がクラスもしくは抽象クラスの場合、コンストラクタ関数型の引数のタプル型を`infer P`で取得し、そのまま`P`を返しています。


### InstanceType

```TS
/**
 * コンストラクタ関数型の戻り値を取得する
 */
type InstanceType<T extends abstract new (...args: any) => any> = T extends abstract new (...args: any) => infer R ? R : any;
``` 

こちらも上記の`ConstructorParameters`と同様に、引数`T`を`extends abstract new (...args: any) => any`でクラスもしくは抽象クラスに制限します。

`T`がクラスもしくは抽象クラスの場合、コンストラクタ関数型の戻り値（クラスのインスタンスの型）を`infer R`で取得し、そのまま`R`を返しています。


間違いなどあればご指摘いただけると幸いです。

### 参照

https://github.com/microsoft/TypeScript/blob/main/src/lib/es5.d.ts

https://www.typescriptlang.org/docs/handbook/utility-types.html#uppercasestringtype

https://www.typescriptlang.org/docs/handbook/advanced-types.html#type-inference-in-conditional-types
