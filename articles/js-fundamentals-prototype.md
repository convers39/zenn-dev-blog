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

`new`演算子でオブジェクト作る時に言及した、`[[Prototype]]`と`prototype`と何が違うのかというと、

- `[[Prototype]]`はオブジェクトの内部属性として、*他のオブジェクト*へのポインターになり、呼び出すには`__proto__`を使います
  - `[[]]`は内部属性のため直接アクセスできないが、`[[Prototype]]`と`__proto__`が同一だと考えて構いません
- `prototype`はとあるオブジェクトを*インスタンス化*するために必要なブループリント・モデルになります
- 新しいオブジェクトを作る際に、上記のブループリントの`prototype`プロパティへ`[[Prototype]]`で紐つきます
  - 例えば`p1`の場合だと、`p1`の`[[Prototype]]`は、`Person.prototype`へ指していて、`p1.__proto__ === Person.prototype`となります

もちろん、同じ仕組みが自作のオブジェクトだけではなく、ビルドインの全てのオブジェクトに存在します。さらに`primitive`タイプの場合は、ラップオブジェクト（`Number`, `String`, `Boolean`など）の`prototype`と繋いでいます。最終的に、全てのオブジェクトが`Object.prototype`と繋ぐことになります。

```js
let arr = [1,2,3];
assert(arr.__proto__ === Array.prototype);

const fn = () => {}
assert(fn.__proto__ === Function.prototype);

const a = 1;
assert(a.__proto__ === Number.prototype); // ES5以前ではエラーになる

const obj = {};
assert(obj.__proto__ === Object.prototype);
// ...
```

図で表現すると、

![](https://storage.googleapis.com/zenn-user-upload/9f622a3cafe1-20240927.png)


### prototype chain

上記の図では多少理解できるかもしれませんが、なぜ`chain`と呼ばられるというと、下記の例でよりわかりやすいくなると思います。

```js
class A {
    sayA {
        console.log('A')
    }
    sayHi {
        console.log('Hi! I am A')
    }
}

class B extends A {
    sayB {
        console.log('B')
    }
    sayHi {
        console.log('Hi! I am B')
    }
}


class C extends B {
    sayC {
        console.log('C')
    }
    sayHi {
        console.log('Hi! I am C')
    }
}
```

上記の例で言えば、`__proto__`の方向は、`C -> B -> A -> Object`になっているわかります。

```js
const a = new A();
const b = new B();
const c = new C();

c.sayC() // c -> sayCはC.prototypeにある
c.sayB() // b -> Bのメソッドも使える
c.sayA() // a -> Aのメソッドも使える
c.sayHi() // Hi! I am C -> 同じメソッドは"上書き"される

assert(A.prototype.isPrototypeOf(a))
assert(a instanceof Object)
assert(b instanceof a)
assert(c instanceof b)
assert(c.__proto__ === C.prototype)
assert(c.__proto__.__proto__ === B.prototype)
assert(c.__proto__.__proto__.__proto__ === A.prototype)
assert(c.__proto__.__proto__.__proto__.__proto__ === Object.prototype)
```

この仕組みで何がすごいかというと、

- インスタンスがポインター`__proto__`で指している`prototype`のプロパティにアクセスできるため、**インスタンスの間にプロパティの共有**ができる
- *継承*する親のメソッドも`__proto__`プロパティを利用し、`prototype chain`に辿って使うことができるため、**子が親からプロパティの継承**ができるだけではなく、**メソッドの上書き**することも可能

もちろん、ここの*上書き*は本当に親クラスのメソッドの上書きというより、prototypeにプロパティが存在する場合優先順位が高く、prototype chain上の同じプロパティへのアクセスが遮られる(shadow)とも言います。この仕組みによって、OOPのポリモーフィズムも実現できるようになります。

プロパティがインスタンスに存在するかどうかは、`Object.hasOwnProperty`メソッドで判定することができます。それに対して、`in`を使う場合はインスタンスに存在しなくても、prototype chainで見つけることができれば判定は`true`になるのです。


### `constructor`とは

そもそも、コンストラクターとはなんだろう。ES6から`class`を使って他のOOP言語のように「クラス」を定義することができるようになりました。例えば、

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

また、`prototype`にメソッドを含めてプロパティを追加することができますが、通常ステートはインスタンスに持たせるため、本当に共有したいプロパティ以外は、コンストラクター関数の中で`this`に付与した方が良いでしょう。

```JavaScript
Person.prototype.name = 'John' // -> 通常はアンチパターン、全てのPersonインスタンスにこの値へのアクセスがあります
```

ちょっと待って、`p1.constructor`はわかったけど、`Person.constructor`はなんなのか、`Person.prototype.constructor`と一緒なのか、どうだろう。

前の節の`prototype chain`を思い出してください。`p1`は`Person`から作ったオブジェクトで、`Person`は関数なので、`Function`というオブジェクトから作られた関数にすぎません。なので、`Person.constructor === Function`となります。

節の冒頭の疑問に戻るが、そもそもconstructorはなんだろうというと、結局関数にすぎません（`p1.constructor === Person`を考えてください）。正しくインスタンスを作るために（`this`を作られたオブジェクトにバインディングするために）`new`を使う必要がある、というくらいの特別要件がありますが、本質的には関数です。そのため、

```js
Person('John') // -> newがない場合はthisがグローバルオブジェクトに紐つく

window.sayHi() // 'Hi! I am John'

let o = {};
Person.call(o, "Doe"); // oにthisが紐つく
o.sayHi() // 'Hi! I am Doe'
```

とのように、関数として使えるし、使い方次第で`this`のバインディングを変えることが可能です（`this`について[こちらの記事](https://zenn.dev/convers39/articles/e0b7a26859f999#%E5%84%AA%E5%85%88%E9%A0%86%E4%BD%8D)に参考してください）。もちろん、ES6の`class`を使う場合は、`new`と併用することが必要になるため基本このようのことは起こりませんが、あくまでも`constructor`の本質を理解するためのハックに過ぎません。

### `class`とコンストラクター関数の違い

よくある言い方として、`class`とはコンストラクター関数のsyntax sugarに過ぎない、と。`class`によって複雑な`prototype`を弄る操作はカプセル化され、他のOOP言語からJSに乗り換える場合は素早く使えるようになります。一方で、両者は完全に一致しているわけではなく、オブジェクトの属性の文脈では少なくとも以下のような違いがあります。

- `enumerable` オブジェクトのプロパティにはイテレート可能かどうかについて`enumerable`で決めていますが、`class`で作られる場合、メソッドは全て`enumerable:false`にはなります。通常これは意図通りの挙動で、`for..in`のループにメソッドを出したい場面がほぼないでしょう。
- `[[IsClassConstructor]]: true` `class`で作られたオブジェクトには`[[IsClassConstructor]]`都の内部属性が存在します。`class`で定義したものを、`new`を使っていない時、内部的に`new.target == undefined`になるため、この内部属性と合わせて`new`なしの作成を禁止しています。
- `use strict` `class`は常に`strict`モードになります。


## 継承と委託

### JSの継承

prototype chainを節には、プロパティの継承と上書き（アクセス遮断）ができるとわかりました。ただこれだけでは継承には成り立ちません。というのは、メソッド以外のプロパティも存在するため、

- 親と子のインスタンスを作る際に別々でステートを保持すること
- 子のインスタンスを作る際に、親にあるステートを継承すること
- インスタンスの間にステートの干渉がないこと

といった条件を満たす必要があります。prototype chainだけでは、仮に`Person.prototype.name = 'John'`をやってしまうと、インスタンス間のステート干渉が問題になってしまいます。

ES6以前では、さまざまなハックテクニックを通してこの問題を解決しようとしていました。

- constructor stealing/borrowing
- combination inheritance (pseudoclassical inheritance) = constructor stealing + prototype chain
- prototypal inheritance
- parasitic inheritance など

ここでは`constructor stealing` + `prototype chain`のやり方を説明します。

```javascript
function Person(name) {
    this.name = name
    this.friends = ['Jay', 'John', 'Jane']
}

Person.prototype.sayHi = function () {
    console.log('Hi, I am ' + this.name)
}

function Engineer(name, lang) {
    Person.call(this, name) // -> Personにthisを渡し、nameを付与する a.k.a constructor stealing/borrowing
    this.lang = lang
}

Engineer.prototype = new Person() // -> Personのメソッドを継承する

Engineer.prototype.sayLang = function () { // -> このタイミングで追加すれば
    console.log('Hi, I am a ' + this.lang + ' engineer' )
}

```

- `Person.call(this, name)`を通して、`Person`の初期化を行い、さらに`Engineer`のインスタンスオブジェクト（`this`）に紐つくことで、`this.friends`のようなレファレンスタイプのプロパティがコピされ、インスタンス間の干渉を避ける
- `Engineer.prototype`に`new Person()`を付与することで、`Person`のプロトタイプと、`Engineer`のプロトタイプと分けると同時に、`Person`にあるメソッドを`Engineer`に使わせる


ES6以降は、`Class`を利用することで継承がだいぶシンプルになりました。`prototype`のアサインとコンストラクター借用の部分は`extends`と`super`で解消されています。本質的には上記のように`prototype`の仕組みを利用しています。

```javascript
class Person {
    constructor(name) {
        this.name = name
        this.friends = ['Jay', 'John', 'Jane']
    }
    // ...
}

class Engineer extends Person { // -> Engineer.prototype = new Person() に相当
    constructor(name, lang) {
        super(name); // -> Person.constructor(name) / Person.call(this, name) に相当
        this.lang = lang;
    }
    // ...
}

```


### OOP言語の継承との区別

実際に、prototypeの方式は*継承*の機能をしているように見えます。継承はOOPの三大柱の一つとしてOOP観点からは重要視されているが、その中心にあるのは、プログラムをより小さいパート（クラス）に分けて、各自のステートを管理し、お互いの通信（messaging）を行うことを通して機能を実現させ、複雑度を下げてより管理可能な構造にすることだと思っています。

JavaScriptにおけるprototype方式の*継承*と他のOOP言語における継承との違いというと、

| 相違点 | 通常のOOP言語 | JSの実現 |
| --------------- | --------------- | --------------- |
| メンタルモデル | クラスとのブループリントからインスタンスを作る | prototype chainで直接他のオブジェクトと繋ぐ |
| オブジェクト作成 | クラスからオブジェクトを作る | コンストラクター関数からオブジェクトを作る |
| インスタンスステート保持 | メソッドやデータ両方保持する | データのみ保持する |
| 継承仕組み | メソッドを含めたプロパティをコピーする | prototypeにあるメソッドを共有する |
| ポリモーフィズム仕組み | 親クラスと同名のメソッドを上書きする | prototype chainにある同名メソッドを実装し、アクセスを遮断する |


特に継承について、親と子という階層の先入観が含まれているが、`Kyle Simpson`氏が曰く、JSでは`OLOO a.k.a. Objects Linked to Other Objects`と読んだ方がふさわしい。

例えば、`arr`を作る際に、別に`fitler`, `map`などのプロパティ定義していないですが、`arr.fitler(...)`は使えるようになります。
これは、`Array.prototype`から*継承*しているようにも見えますが、厳密に言えば継承の枠組みではなく、言葉で表現すると代用・代理に近いかもしれん（公式の言葉では次節の委託delegationになる）。例えば、

```js
let obj = {
    0: 'a',
    1: 'b',
    length: 2
}

obj.join = Array.prototype.join
// もしくは
obj.__proto__ = Array.prototype

console.log(obj.join(',')) // 'a, b'
console.log(Array.isArray(obj)) // false, obj.__proto__ = Array.prototypeの場合ならtrue
```

というようなことができます。`obj`は`Array`を*継承*しているかどうかと言われると、他のOOP言語の視点からNOかと思います。objはオブジェクトのままで配列のメソッドを借りているので、厳密な誰が親か誰が子かの関係はありません。このようなオブジェクト間のメソッド借用が明らかに継承の枠組みを超えていると思います。

### 委託組み合わせ（delegation composition）

さて、JSの継承について少しみてきましたが、実は継承について、よく`composition over inheritance`と言われています。コンポジションまたは組み合わせについて[別の記事](https://zenn.dev/convers39/articles/83dd5898d4d798#%E7%B6%99%E6%89%BF)で少し書いたことがありますが、シンプルにいうと、`is-a`の継承関係を求めずに、`a-able`という、機能性を求める観点からのコード再利用となります。特に、JSの場合はマルチ継承ができないため、仮に複数のオブジェクトから借用したい時に上記の継承パターンが無力になります。

ここでJSの`prototype`の特性に合わせた組み合わせ（delegation composition）について紹介します。ここでウェブアプリ開発のロールを例に考えてみてください。

```javascript
// 開発者のオブジェクト
const developer = {
    name: '',
    sayName() {
        console.log(`Hi, I am ${this.name}`);
    }
};

// フロントエンド開発に必要なスキル
const frontendSkills = {
    buildUI() {
        console.log(`${this.name} is building a beautiful UI!`);
    }
};

// バックエンド開発に必要なスキル
const backendSkills = {
    buildAPI() {
        console.log(`${this.name} is building a powerful API!`);
    }
};

// フロントエンドエンジニア
const frontendDev = Object.assign(Object.create(developer), frontendSkills);
frontendDev.name = 'Alice';
frontendDev.sayName(); // "Hi, I am Alice"
frontendDev.buildUI(); // "Alice is building a beautiful UI!"

// バックエンドエンジニア
const backendDev = Object.assign(Object.create(developer), backendSkills);
backendDev.name = 'Bob';
backendDev.sayName(); // "Hi, I am Bob"
backendDev.buildAPI(); // "Bob is building a powerful API!"

// フルスタックエンジニア
const fullstackDev = Object.assign(Object.create(developer), frontendSkills, backendSkills);
fullstackDev.name = 'Charlie';
fullstackDev.sayName(); // "Hi, I am Charlie"
fullstackDev.buildUI(); // "Charlie is building a beautiful UI!"
fullstackDev.buildAPI(); // "Charlie is building a powerful API!"
```

ここは`Developer`から`FrontendDev`、`BackendDev`が継承するようなイメージではなく、`a-able`の機能性の部分（スキル）を抽出しています。`Object.assign`する際に、まず`Object.create`によって`__proto__`を`developer.prototype`へ紐つきます。すると`developer.sayName`が共有されるようになります。フルスタックエンジニアを考える時に、当然FEとBE両方のスキルが必要なので、継承の考え方だと結構厄介なケースになりますが、上記のようにスキルを組み合わせることで簡単に達成できます。

組み合わせのパターンについて他の言語には`trait`, `mixins`といったものが相当します。`interface`の場合はステートや実装を保持しない点で`trait`, `mixins`から少し離れますが、機能性の部分を抽出して組み合わせる考え方自体は`interface`にも運用可能です。

## 実運用の注意点

### `prototype`汚染(pollution)

prototypeの性質を利用することで攻撃者が`Object.prototype`とかを弄ることで、プログラム内の全てのオブジェクトを影響することが可能です。例えば、

```javascript
Object.prototype.isAdmin = true;

const user = {};
console.log(user.isAdmin); // true
```

これは実際にユーザーインプットから起こる可能性があります。`JSON.parse`を行う時に不用意にやられることが考えられると思われます。

```javascript
const maliciousPayload = JSON.parse('{"__proto__": {"isAdmin": true}, "userId": 1}');

const currentUser = await db.getUser(maliciousPayload.userId)
const updatedUser = {...currentUser, ...maliciousPayload};
```

これを防止するために、いくつかの考え方があります。

- コード上は`Object.prototype`のような変更を禁止する（基本当たり前）
- `__proto__`プロパティのないオブジェクトを作成する（`Object.create(null)`）
- ユーザーインプットをsanitizeする（例：`obj.hasOwnProperty('__proto__') || obj.hasOwnProperty(constructor)`）
- オブジェクト操作を外部ライブラリから行う場合、`prototype`汚染に対策しているライブラリーを使う

### 意図しない`prototype`の変更

`Object.prototype`の変更は全てのオブジェクトに影響するため、一番避けたいパターンではあります。それと近い考え方で、ビルトインのオブジェクトの`prototype`も必要がない限りいじらない方が良いでしょう。

ここで主に2つの考え方があります。

- ビルトインから拡張のクラスを作る、例えば

```javascript
// ❌
Array.prototype.last = function() {
    return this[this.length - 1];
};

// ✅
class MyArray extends Array {
    last() {
        return this[this.length - 1];
    }
}

const arr = new MyArray(1, 2, 3);
console.log(arr.last()); // 3

```

- どうしても`prototype`を弄る場合は、シンボルを利用して潜在的な競合可能性（将来公式で同じAPIが実装されたとか）を避ける、例えば

```JavaScript
const lastSymbol = Symbol('last');
Array.prototype[lastSymbol] = function() {
    return this[this.length - 1];
};

const arr = [1, 2, 3];
console.log(arr[lastSymbol]()); // 3
```

### パフォーマンス

オブジェクトに必要なプロパティがみつからない場合、該当オブジェクトのprototype chain    に辿って探索を行うので、探索のパフォーマンスで言えばこの木の深さと比例します。

正直これで困る場面がまだ会っていませんが、一応対策の考え方として

- 継承というより組み合わせを活用する
- `hasOwnProperty`を利用して不要な探索を止める

## まとめ
