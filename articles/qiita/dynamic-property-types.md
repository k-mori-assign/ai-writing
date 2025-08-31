---
title: 【TypeScript】動的なプロパティをもつオブジェクトの型を定義する
tags: TypeScript JavaScript
author: kikikikimorimori
slide: false
---
### はじめに
TypeScriptで、一部のプロパティだけが動的に変わるようなオブジェクトの型定義について、`Mapped Types`, `インデックスシグネチャ`, `インターセクション型`を用いて定義してみました。

最終的なコードは下記になります。

```ts
type FixedProps = {
  [key in 'prop1' | 'prop2' | 'prop3']: { value: string }
}

type DynamicProps = {
  [key: string]: {
    value1?: string,
    value2?: number,
  }
}

type Props = FixedProps & DynamicProps
```

### Mapped Typesとは

`Mapped Types`は、ある型をベースに新しい型を作る機能です。

`Required`や`Omit`などの`utility type`の内部でも使われています。

例えば、`Required`の定義は下記のようになっています。

```ts
type Required<T> = {
    [P in keyof T]-?: T[P];
};
```

引数で渡される型`T`のプロパティが、`keyof`と`in`によって一つずつ型変数`P`に入り、`-?`でオプショナルを外し、`T[P]`によって該当のプロパティの値の型を再定義しています。

定義を見て少し思ったのが、引数`T`に対して`extends`で`{ [key: string]: any }`とか`object`を指定して、引数をオブジェクトのみに限定するのもアリなのかなと思いました。

```ts
type Required<T extends { [key: string]: any }> = {
    [P in keyof T]-?: T[P];
};

// Type 'string' does not satisfy the constraint '{ [key: string]: any; }'
type sample = Required<string>
```

少し話しが逸れましたが、今回は固定のプロパティの部分で、プロパティの値の型が同じであり、繰り返しの記述を避けるため使いました。

下記の部分になります。

```ts
type FixedProps = {
  [key in 'prop1' | 'prop2' | 'prop3']: { value: string }
}
```

`in`によってユニオン型の値が1つずつ取り出され`key`として定義されます。

最終的には下記のような型になります。

```ts
type FixedProps = {
    prop1: { value: string },
    prop2: { value: string },
    prop3: { value: string }
}
```


### インデックスシグネチャとは
インデックスシグネチャは、プロパティ名を指定せず、プロパティ名の型と値の型のみを指定する機能です。

例えば、`key`が`string`型で値がすべて`number`型の定義は下記のようになります。

```ts
type IndexSignature = {
    [key: string]: number
}
```

`key`は型変数なので任意の名前をつけられますが、`key`や`K`で定義されることが多いです。

また、`key`の型には、`string`, `number`, `symbol`, `テンプレートリテラル型`の4つを指定できます。

例えば、テンプレートリテラル型を使って`sample_`ではじまるプロパティだけに制限する場合は下記のようになります。

```ts
type TemplateLiteralPropType = { [key: `sample_${string}`]: string };

const sampleObj: TemplateLiteralPropType = {
  sample_key: 'a',
  sample_key2: 'b',
  key3: 'c',  //  Object literal may only specify known properties, and 'key3' does not exist in type 'TemplateLiteralPropType'.
}
```

最初に定義した一部動的なプロパティをもつオブジェクトの型においては、動的に変わるプロパティの部分で使用しています。

```ts
type DynamicProps = {
  [key: string]: {
    value1?: string,
    value2?: number,
  }
}
```

この場合、プロパティの`key`の型が`string`で、値が`value1`と`value2`という任意のプロパティを持つオブジェクトになります。

こうすることで、動的なプロパティの値を使いたい時などに型推論が効くようになります。

下記の変数`sample`の型が上記の`DynamicProps`の場合
![スクリーンショット 2022-11-05 19.07.42.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1248963/a6a87e2c-7ea9-d975-033a-f0cbb33412f2.png)


### インターセクション型とは
インターセクション型は、型同士をまとめて新しい型を作る機能です。

##### オブジェクト型同士の場合
オブジェクト型に対してインターセクション型を使う場合は、それぞれのオブジェクトのプロパティをもつ新たな型が定義されます。

```ts
type Obj1 = {
  a: string,
  b: number,
}

type Obj2 = {
  c: boolean,
}

type IntersectionType = Obj1 & Obj2

// 下記のようになる
// type IntersectionType = {
//   a: string,
//   b: number,
//   c: boolean,
// }

```

##### プリミティブ型同士の場合

異なるプリミティブ型に対してインターセクション型を使うと、`never`型になります。

```ts
type Primitive = string & number

// type Primitive = never
```

##### プリミティブ型で構成されたユニオン型同士の場合

プリミティブ型で構成されたユニオン型同士に対してインターセクション型を使うと、共通するプリミティブ型のみの型になります。

```ts
type PrimitiveUnionType1 = string | number
type PrimitiveUnionType2 = string | boolean

type IntersectionType = PrimitiveUnionType1 & PrimitiveUnionType2

// type IntersectionType = string
```

これは一つ前で説明したプリミティブ型同士のインターセクション型は`never`型になるという性質のためです。

上記の場合、`PrimitiveUnionType1`と`PrimitiveUnionType2`の組み合わせは下記の4通りです。

```js
string & string   // => string
string & boolean  // => never
number & string   // => never
number & boolean  // => never
```

上記のように、プリミティブ型で構成されたユニオン型同士に対してインターセクション型を使う場合、共通するプリミティブ型以外の場合は`never`型になるため、共通するプリミティブ型のみの型になります。

### 一部動的に変わるプロパティを持つオブジェクトの型定義

以上の`Mapped Types`, `インデックスシグネチャ`, `インターセクション型`を用いて一部のプロパティだけが動的に変わるようなオブジェクトの型定義をしてみました。

```ts
type FixedProps = {
  [key in 'prop1' | 'prop2' | 'prop3']: { value: string }
}

type DynamicProps = {
  [key: string]: {
    value1?: string,
    value2?: number,
  }
}

type Props = FixedProps & DynamicProps
```

もっと良い方法や間違いなどあればご指摘いただけると幸いです!
