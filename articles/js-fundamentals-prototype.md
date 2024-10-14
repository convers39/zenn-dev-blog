---
title: "JS基礎いろいろーPrototype"
emoji: "⛓️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript"]
published: false
---

## クイズ

## prototypeとは何か

### オブジェクト

prototypeを理解するためには、まずオブジェクトのことを理解しなければなりません。JSのデータタイプ的には、primitiveタイプと、objectタイプに分けられることができます（[参考](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures) ）。オブジェクトタイプのデータはシンプルに言えば、キーとバリューのペアで構成されたプロパティの集合（collection）として考えられます。

```ts
const person = {
    name: 'John',
    sayHi() {
        console.log('Hi! I am ' + this.name);
    }
};
```

プロパティの集合で考える時に、JSには2種類のプロパティタイプを定義しています。プロパティの挙動の定義はJS内部的な属性で決められていて、通常`[[xxx]]`のように囲まれています。

#### data properties

このプロパティは、バリューの読み書きの場所（location）とその挙動を定義しています。

- `[[Configurable]]` `delete`キーワードで削除されたり、プロパティの内部属性が変更可能かどうかの行為を定義している内部属性。デフォルトはtrue。
- `[[Enumerable]]` シンプルに言えば、オブジェクトに対してループをかける際に、ループのリターン値（ループ対象）になるかどうかを定義。デフォルトはtrue。
- `[[Writable]]` 値（value）が変更可能かどうか。デフォルトはtrue。
- `[[Value]]` 実際の値が保存されている場所。デフォルトは`undefined`

これらの内部属性は、`Object.defineProperty`の静的メソッドで定義されることが可能です。

```ts
let person = {};
Object.defineProperty(person, "name", {
    writable: false, // リードオンリーになる
    enumerable: false, // for inの時にnameのキーが出なくなる
    configurable: false, // 再度Object.definePropertyでnameプロパティの属性変更できなくなる
    value: 'John'
})
```

もちろん、普段の業務でここまで使うことがないでしょうが、JSのオブジェクトを理解するためには知っておいた方が良いかなと思います。

#### accessor properties

accessorは名前通り、データのアクセス（とセット）が関心となります。内部属性はデータプロパティと多少違います。

- `[[Configurable]]` 一緒
- `[[Enumerable]]` 一緒
- `[[Get]]` プロパティがreadされる際に呼び出す関数、デフォルトは`undefined`
- `[[Set]]` プロパティがwriteされる際に呼び出す関数、デフォルトは`undefined`

上記のデータプロパティと同じく、`Object.defineProperty`で定義することが可能です。

```ts
let person = {
    name: 'John',
    age_: 30,
};

Object.defineProperty(person, "age", {
    get() {
        return this.age_;
    }
    set(newValue) {
        if (newValue < 0) {
            throw new Error('Are you sure?'):
        }
        if (newValue > 200) {
            throw new Error('Not a human!'):
        }
        this.age = newValue
    }
})
```

また、上記の例で定義されたプロパティの属性は`Object.getOwnPropertyDescriptor`で確認することができます。

なぜ、`OWN`が強調されるかは、一旦伏せておいて、後の節で自明になります。

### オブジェクトを作る

オブジェクトを作るには、コンストラクター関数や、`Class`, `Object.assign`, `Object.create`, `{}`で付与するなど、様々な方法があります。

コンストラクター関数はECMA6以前によく使われる方法です。

```js
function Person(name) {
    this.name = name;
    this.sayHi = function() {
        console.log('Hi! I am ' + this.name);
    }
}

// newで作成
let person = new Person(...)
```

`new`演算子を使っている時に何が起こっているかというと、

- 新しいオブジェクトがメモリーに作られます
- 新しいオブジェクトのポインター`[[Prototype]]`が、コンストラクター関数の`prototype`プロパティへ紐づきます
- `this`が新しいオブジェクトにアサインされます
- コンストラクター関数の中身が実行されます

つまり、コードで言えば以下のコメントが暗黙的に行ったわけです。

```js
function Person(name) {
    // this = {};
    this.name = name;
    this.sayHi = function() {
        console.log('Hi! I am ' + this.name);
    }
    // return this;
}

// 以下は同じ効果
let p1 = new Person('p1');
let p2 = {name; 'p2'};
```

### `[[Prototype]]`と`prototype`

`new`演算子でオブジェクト作る時に言及した、`[[Prototype]]`と`prototype`と何が違うのか

- 前者はオブジェクトの内部属性として、*他のオブジェクト*へのポインターになり、呼び出すには`__proto__`を使います（`[[]]`は内部属性のため直接アクセスできない）
- 後者はとあるオブジェクトを*インスタンス化*するために必要なブループリント・モデルになります
- *他のオブジェクト*というのは、まさに上記のブループリントのことです
- 例えば`p1`の場合だと、`p1`の`[[Prototype]]`は、`Person.prototype`へ指していて、`p1.__proto__ === Person.prototype`となります

これって、なんの意味があるのか、と言われると、同じ仕組みが自作のオブジェクトだけではなく、ビルドインの全てのオブジェクトに存在するので、この`[[Prototype]]`のポインターによって全てのオブジェクトは実は繋いていることになります。

例えば、

```js
let arr = [1,2,3];
assert(arr.__proto__ === Array.prototype);

const fn = () => {}
assert(fn.__proto__ === Function.prototype);

const a = 1;
assert(a.__proto__ === Number.prototype);

const obj = {};
assert(obj.__proto__ === Object.prototype);
// ...

assert(Function.prototype.__proto__ === Object.prototype)
assert(Person.prototype.__proto__ === Object.prototype)
assert(Array.prototype.__proto__ === Object.prototype)
// ...
```

![](https://storage.googleapis.com/zenn-user-upload/9f622a3cafe1-20240927.png)

シンプルにまとめると、

- 全てのビルトインのオブジェクトには、メソッドをprototypeに紐ついている
  - primitiveタイプの場合は、ラップオブジェクト（Number, String, Booleanなど）のprototypeに頼っている
- オブジェクト（もしくはインスタンス）はデータだけを保存する
- インスタンスから`__proto__`プロパティを経由して、オブジェクトのprototypeに紐つくメソッドをアクセスし、同じメソッドをシェアすることが可能


この仕組みで何がすごいかというと、ポインターで指しているプロトタイプのプロパティはアクセスできるからです。例えば、`arr`を作る際に、別に`fitler`, `map`などのプロパティ定義していないですが、`arr.fitler(...)`は使えるようになります。

これは、`Array.prototype`から*継承*しているようにも見えますが、厳密に言えば継承の枠組みではなく、言葉で表現すると代用・代理に近いかもしれん。例えば、

```js
let obj = {
    0: 'a',
    1: 'b',
    length: 2
}

obj.join = Array.prototype.join
// もしくは、obj.__proto__ = Array.prototype

console.log(obj.join(',')) // 'a, b'
```

というようなことができます。`obj`は`Array`を*継承*しているかどうかと言われると、他のOOP言語の視点からNOかと思います。JSのprototypeを活用することで、オブジェクト間のメソッド借用が可能になります。これは明らかに継承の枠組みを超えていると思います。


### `constructor`とは

そもそも、コンストラクターとはなんだろう。ES6から`class`を使って他のOOP言語のように「クラス」を定義することができるようになりました。

例えば、

```js
class Person {
    constructor(name) {
        this.name = name
    }
    sayHi(){
        console.log('Hi! I am ' + this.name)
    }
}
```

先ほどのコンストラクター関数と実質同じで、`new Person('a')`を行う時に、`Person`という**関数**を`new`で呼び出し、`constructor`メソッドに定義されているものが実行されます。クラスに存在するメソッドは、`Person.prototype`に紐つくようになります。そのため、JSの`class`何か新しい仕組みを作っているわけではなく、根底にはオブジェクトの作成になっています。

それで、`constructor`メソッドも同じく、`Person.prototype`に紐ついています。これは、全ての関数の`prototype`にデフォルト値として、自分自身に指すオブジェクトとなっているからです。


```js
console.log(typeof Person) // function
console.log(Person === Person.prototype.contructor) // true
// つまり
// Person.prototype = { constructor: Person }
```

インスタンス化する際に、インスタンスの`constructor`プロパティが、クラスの関数となります。

```js
const p1 = new Person('1')
console.log(p1.constructor === Person) // true　ただ通常はinstanceofを使う
const p2 = new p1.constructor('2') // も一応可能
```

これらの関係を図で表現すると

![](https://storage.googleapis.com/zenn-user-upload/ac5994cd53a4-20241007.png)

ただ要注意したいのは、デフォルトの`constructor`が、関数の`prototype`と紐ついていますが、これは`writable`なプロパティなので、入れ替えることが可能です。もし`Person.prototype = {}`とかにしてしまうと、デフォルトの`constructor`がなくなります。`prototype`を仮にいじる必要がある時は、破壊変更になるかどうかは気をつけた方が良いでしょう。

ちょっと待って、`p1.constructor`はわかったけど、`Person.constructor`はなんなのか、`Person.prototype.constructor`と一緒なのか、どうだろう。

前の節の`prototype chain`を思い出してください。`p1`は`Person`から作ったオブジェクトで、`Person`は関数なので、`Function`というオブジェクトから作られた関数にすぎません。なので、`Person.constructor === Function`となります。`prototype chain`の意味で言えば、*クラス*や*インスタンス*との概念がJSには合わない側面があり、このようにクラスが同時にインスタンスにもなっている結論になってしまうからです。

節の冒頭の疑問に戻るが、そもそもconstructorはなんだろうというと、結局関数にすぎません（`p1.constructor === Person`を考えてください）。正しくインスタンスを作るために（`this`を作られたオブジェクトにバインディングするために）`new`を使う必要がある、というくらいの特別要件がありますが、本質的には関数です。そのため、

```js
Person('John') // -> newがない場合はthisがグローバルオブジェクトに紐つく

window.sayHi() // 'Hi! I am John'

let o = {};
Person.call(o, "Doe"); // oにthisが紐つく
o.sayHi() // 'Hi! I am Doe'
```

とのように、関数として使えるし、使い方次第で`this`のバインディングを変えることが可能です（`this`について[こちらの記事](https://zenn.dev/convers39/articles/e0b7a26859f999#%E5%84%AA%E5%85%88%E9%A0%86%E4%BD%8D)に参考してください）。無論、`new`と併用することがわかりやすくておすすめです。

### `class`とコンストラクター関数の違い

- `enumerable`

- `[[IsClassConstructor]]: true`

- `use strict`


## 継承と代理

### prototypeが解決したい問題

### 継承、組合せとの区別

## 実運用の注意点

### prototype pollution

### `instanceof`

## まとめ
