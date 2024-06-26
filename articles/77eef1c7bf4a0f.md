---
title: "Node.js + MySQL 全文検索で入力サジェスト候補を出す"
emoji: "🎆"
type: "tech"
topics:
  - "javascript"
  - "mysql"
  - "nodejs"
  - "typescript"
published: true
published_at: "2022-03-18 22:41"
---

### 機能リクエスト

最近業務で、インプットで入力する際にサジェスト機能で候補を出してほしい、とのリクエストがありました。

フロントエンドはReactとreact-selectで実装していて、入力＋候補を出すところが割と簡単にできました。今回はそのバックエンドの実装についてまとめます。

### テーブル構造

最低限の情報として、候補の名前が欲しいですね。

ただ、日本語で候補を出す時に考えないといけないのが、

- 入力文字が漢字なのか、仮名なのか、英数字なのかなど
- 仮に仮名またローマ字で入力する場合、漢字の候補を出すかどうか

英語のようなシンプルに英数字で入力して候補を検索すれば良い、という一筋にはいかないですね。UX的に、仮名であろうと、ローマ字であろうと、**発音**が同じ・近いものを出した方が、無駄に新規のデータを作らずに、「これ賢いな」との錯覚を与えることができます。

発音を基準に検索する、のであれば、名前のフィールド以外に、発音のフィールドも欲しいとのことですね。するとこちらのフィールドを導入しました。

- name varchar(200)
- romaji varchar(400)

### 検索の問題点

次に検索のSQLですが、まずはLIKEで考えるかもしれません。ただ、LIKEで検索するときに、少し問題があります：

- `LIKE a%` -> 先頭一致、aから始まる名前のデータが対象になります。普通のインデックスが機能します。
- `LIKE %a` -> 後尾一致、aが最後のデータが対象になります。普通のインデックスが機能しません。
- `LIKE %a%` -> 部分的に一致、aが現れれば対象になります。普通のインデックスが機能しません。

MySQLのインデックスは通常B-TREEのデータ構造で作られています。B-TREEは例えで言えば、電話帳、辞書（電子辞書のことではなく紙のもの）のようなもので、先頭から調べるのが想定されているので、先頭一致だと非常に早いです。しかし、逆のパターンになると、辞書で最後が「ご」の言葉を調べるように難しいことになります。

もちろん、もう一つのフィールドを追加して、データ保存すると同時に、abcをcbaに逆さまに保存することで、擬似的に`LIKE %a`にもインデックスを使えるように出来ますが、結局部分的に一致のケースはどうしようもありません。

今回のリクエストで考えると、先頭一致だけであれば、LIKE + B-TREEでなんとかなりそうですが、もし今後部分検索もしたい、とのリクエストが出ると、LIKEのままでスピードがだいぶ落ちます。それを考えると、やはりここは全文検索の出番になるかな、との判断です。

### 全文検索

全文検索（Full-text search）はなんか難しそうな言葉ですが、元々、膨大なテキストデータから**とある特定のターゲット情報**を見つけ出す、との技術の総称です。それを素早く行うために、テキストデータを一定の基準で細かいパーツ（トークン・token）に分解して、その分解されたパーツに対してインデックスを作ることで、パーツだけの部分的な情報で結果を見つけ出すことが可能になります。

MySQLのngram全文パーサーで例をあげれば、例えばFrankというテキストに対して下記のように分解し、どれを入力しても、Frankが検索結果に上がります。

```
"Frank" => [
  "f", "fr", "fra", "fran", "frank",
  "r", "ra", "ran", "rank",
  "a", "an", "ank",
  "n", "nk",
  "k"
]
```

もちろん、細かく分解することによって、検索の効率が上がりますが、インデックス自身のサイズも膨大になりがちです。という時に、トークン（パーツ）のサイズを調整することができます。

```
// abcdに対して違うトークンサイズ
n=1: 'a', 'b', 'c', 'd'
n=2: 'ab', 'bc', 'cd'
n=3: 'abc', 'bcd'
n=4: 'abcd'
```

#### パーサーとトークンサイズ

MySQLのデフォルトトークンサイズは3です。設定変数は以下となります。

```
[mysqld]
innodb_ft_min_token_size=2 // InnoDB
ft_min_word_len=2 // MyISAM
```

ただ、ngramパーサーを使う場合、上記とは別の設定がありますが、デフォルトサイズが2です。MySQLのデフォルトのパーサーだと、スペースや半角カンマなどを目印に単語の切り目を見ていますが、日本語だと役に立ちません。ngramはCJK(中国語、日本語、韓国語)のためのパーサーで、この欠陥を補っていますので、CJK言語にはngramパーサーが使用されると想定して問題ないでしょう。

```
[mysqld]
ngram_token_size=2 // ngramパーサー専用
```

今回はローマ字対象に検索し、典型的な日本語の一文字がほとんど2つのローマ字からなっているので、デフォルトのままで構いません。ngramパーサーは必須なのかと疑っていましたが、結局日本語で入力するから、ローマ字に変換してもスペースとかの区切りがないので、今回はngramパーサーのままにします。

```sql
-- 新規作成
CREATE TABLE articles (
      id INT UNSIGNED AUTO_INCREMENT NOT NULL PRIMARY KEY,
      title VARCHAR(200),
      body TEXT,
      FULLTEXT (title,body) WITH PARSER ngram
    ) ENGINE=InnoDB CHARACTER SET utf8mb4;
-- 既存テーブルに追加
ALTER TABLE suggestions ADD FULLTEXT INDEX ngram_idx (romaji) WITH PARSER ngram;
```

#### 自然言語モード

MySQLの全部検索にはいくつかのモードがあります。自然言語モードでは、検索結果とキーワードの間の関連度合い（relevance）を基準に降順に並びます。このモードでは関連度合いを表すスコア（score）の取得が可能です。

以下は公式の例です：

```SQL
mysql> CREATE TABLE articles (
          id INT UNSIGNED AUTO_INCREMENT NOT NULL PRIMARY KEY,
          title VARCHAR(200),
          body TEXT,
          FULLTEXT (title,body)
        ) ENGINE=InnoDB;
Query OK, 0 rows affected (0.08 sec)

mysql> INSERT INTO articles (title,body) VALUES
        ('MySQL Tutorial','DBMS stands for DataBase ...'),
        ('How To Use MySQL Well','After you went through a ...'),
        ('Optimizing MySQL','In this tutorial, we show ...'),
        ('1001 MySQL Tricks','1. Never run mysqld as root. 2. ...'),
        ('MySQL vs. YourSQL','In the following database comparison ...'),
        ('MySQL Security','When configured properly, MySQL ...');
Query OK, 6 rows affected (0.01 sec)
Records: 6  Duplicates: 0  Warnings: 0

mysql> SELECT * FROM articles
        WHERE MATCH (title,body)
        AGAINST ('database' IN NATURAL LANGUAGE MODE);
+----+-------------------+------------------------------------------+
| id | title             | body                                     |
+----+-------------------+------------------------------------------+
|  1 | MySQL Tutorial    | DBMS stands for DataBase ...             |
|  5 | MySQL vs. YourSQL | In the following database comparison ... |
+----+-------------------+------------------------------------------+
2 rows in set (0.00 sec)

mysql> SELECT id, MATCH (title,body)
    AGAINST ('Tutorial' IN NATURAL LANGUAGE MODE) AS score
    FROM articles;
+----+---------------------+
| id | score               |
+----+---------------------+
|  1 | 0.22764469683170319 |
|  2 |                   0 |
|  3 | 0.22764469683170319 |
|  4 |                   0 |
|  5 |                   0 |
|  6 |                   0 |
+----+---------------------+
6 rows in set (0.00 sec)
```

#### ブールモード

ブールモードでは、AND, ORなどの関係を元に、キーワード検索を行います。検索結果は自然言語モードとは違い、並び替えされませんので、自然言語モードと結果が同じでも順番が異なることがあります。

```SQL
mysql> SELECT * FROM articles WHERE MATCH (title,body)
    AGAINST ('+MySQL -YourSQL' IN BOOLEAN MODE);
+----+-----------------------+-------------------------------------+
| id | title                 | body                                |
+----+-----------------------+-------------------------------------+
|  1 | MySQL Tutorial        | DBMS stands for DataBase ...        |
|  2 | How To Use MySQL Well | After you went through a ...        |
|  3 | Optimizing MySQL      | In this tutorial, we show ...       |
|  4 | 1001 MySQL Tricks     | 1. Never run mysqld as root. 2. ... |
|  6 | MySQL Security        | When configured properly, MySQL ... |
+----+-----------------------+-------------------------------------+
```

一部の演算子として：

- `+`: `+hoge`とは、`hoge`が必ず結果に含まれること
- `-`: `-hoge`とは、`hoge`が必ず結果に含まれないこと
- `なし`: `hoge`とは、`hoge`が必須ではないが、含まれると優先される
- `~`: `~hoge`とは、`hoge`が邪魔なのでこれがあると関連度合いを下げたい
- `*`: `h*`とは、`h`から始まるトークンを対象にし、長さは1として認識される

他の詳細は[公式ガイド](https://dev.mysql.com/doc/refman/8.0/ja/fulltext-boolean.html)にご参照ください。

#### クエリー拡張モード（Full-Text Searches with Query Expansion）

言葉だけでは訳わからないかもしれません。例を見た方が早いかと：

```SQL
mysql> SELECT * FROM articles
    WHERE MATCH (title,body)
    AGAINST ('database' IN NATURAL LANGUAGE MODE);
+----+-------------------+------------------------------------------+
| id | title             | body                                     |
+----+-------------------+------------------------------------------+
|  1 | MySQL Tutorial    | DBMS stands for DataBase ...             |
|  5 | MySQL vs. YourSQL | In the following database comparison ... |
+----+-------------------+------------------------------------------+
2 rows in set (0.00 sec)

mysql> SELECT * FROM articles
    WHERE MATCH (title,body)
    AGAINST ('database' WITH QUERY EXPANSION);
+----+-----------------------+------------------------------------------+
| id | title                 | body                                     |
+----+-----------------------+------------------------------------------+
|  5 | MySQL vs. YourSQL     | In the following database comparison ... |
|  1 | MySQL Tutorial        | DBMS stands for DataBase ...             |
|  3 | Optimizing MySQL      | In this tutorial we show ...             |
|  6 | MySQL Security        | When configured properly, MySQL ...      |
|  2 | How To Use MySQL Well | After you went through a ...             |
|  4 | 1001 MySQL Tricks     | 1. Never run mysqld as root. 2. ...      |
+----+-----------------------+------------------------------------------+
6 rows in set (0.00 sec)
```

上記の公式の例ですが、1回目の検索では、`database`キーワードに対して、マッチするレコードが2つだけでした。しかし、2回目の検索では、仮に`database`とのキーワードがなくても、1回目の結果に基づき、`MySQL`のあるレコードには`database`もある、という結果から推測し、2回目の結果では`MySQL`を含むレコードも対象となりました。つまり、1回目の結果をもとに、`database`とのキーワードと関連度が高い`MySQL`との言葉を抽出し、2回目の検索でそれを射程に入れたとのことです。

ただ、これを実現するには、2回検索が必要なのですので、実行効率といえば前の2つより劣ります。長いテキスト、複数のカラムに対して検索する場合、関連性のあるデータが全部欲しい、との場面に適しているかと思われますが、今回のように短い名前とかにあまり向いていません。そのため、今回は実装の時にこちらのモードを選択肢から外しました。

### 日本語解析ツール

次に考えるのは、日本語のテキストを、発音を表すローマ字に解析することです。このローマ字に対して全文検索インデックスをつけます。

バックエンドがNode.jsなので、既存のパッケージを探したら、[kuroshiro](https://github.com/hexenq/kuroshiro)というのがありました。データのアナライザーとしていくつか選択肢がありますが、一番スター数の多い[kuromoji](https://github.com/hexenq/kuroshiro-analyzer-kuromoji)を採用。

使い方は公式にもありますが、

```js
import Kuroshiro from "kuroshiro";
// Initialize kuroshiro with an instance of analyzer (You could check the [apidoc](#initanalyzer) for more information):
// For this example, you should npm install and import the kuromoji analyzer first
import KuromojiAnalyzer from "kuroshiro-analyzer-kuromoji";
// Instantiate
const kuroshiro = new Kuroshiro();
// Initialize
// Here uses async/await, you could also use Promise
await kuroshiro.init(new KuromojiAnalyzer());
// Convert what you want
const result = await kuroshiro.convert("感じ取れたら手を繋ごう、重なるのは人生のライン and レミリア最高！", { to: "hiragana" });
```

これで実装してみると、いくつか問題がありました。

#### tsのタイプがない問題

よしこれだ！と決めたところ、tsを使っているのですが、これらのパッケージにはタイプ宣言ファイルがありません。

ただ運よく、他の方が[PR](https://github.com/hexenq/kuroshiro/pull/93/files)で作っていたので（メンテされていないせいかマージされていませんが）、それをコピーして追加すれば解消。

設定方法について、tsconfigファイルに、typeRoot項目に自作タイプ宣言ファイルのフォルダーを追加し、excludeにコンパイル対象から除外すれば良いです。例えば、ルートフォルダーに`@types`フォルダーを作って自作タイプ宣言ファイルを入れるとします：

```json
{
  "compilerOptions": {
    //...
    "typeRoots": ["./@types", "./node_modules/@types"]
  },
  "exclude": ["node_modules", "./@types"]
}
```

#### 一文字で検索するとヒットしない問題

デフォルトのトークンサイズ2で実装しているので、一文字で検索するとヒットする対象がありません。もちろん、トークンサイズを1に調整すればいけますが、インデックスサイズが膨大すぎる恐れがあるのでここはバランスをとりたいです。

という時に、ワイルドガードの`*`を使うことで解消できます。こちらは[公式ガイド](https://dev.mysql.com/doc/refman/8.0/ja/fulltext-search-ngram.html)にも記載あります。

#### 一部の検索テキストが無視される問題

これはMySQLのデフォルトのストップワードテーブルと関わっています。ストップワードが検索時に無視されます（[こちら](https://dev.mysql.com/doc/refman/8.0/ja/fulltext-natural-language.html)に参照）。このリスト自体は英単語ですが、例えばtoが入ったりして、ローマ字のtoで検索すると結果がない羽目になります。

これを解消するために、ストップワード機能をオフにするか、別途自作のストップワードテーブルを作るか、との解決法があります。

- ストップワードをオフにする（[こちら](https://qiita.com/ytyng/items/3c936caba6bcfe782614)にも参照）

```
[mysqld]
innodb_ft_enable_stopword = OFF
```

- 自作テーブルを使う（空なので無視される言葉がありません）

```
CREATE TABLE IF NOT EXISTS stopwords (value varchar(255) NOT NULL PRIMARY KEY);
SET GLOBAL innodb_ft_server_stopword_table = '<db_name>/stopwords';
```

#### 初期化が重い問題

`kuroshiro.init(...)`との操作は、十数メガバイトのデータをロードするため、1秒以上かかります。毎回これをやると使い物になりません。

これをなんとか解決しないとですね。

### シングルトンパーサー

kuroshiroの初期化が重いので、どうにかキャッシングして使いまわした方が良い、との考えでした。

という時に、やはりシングルトンのパターンが一番適しているのではないかと。一回初期化されたら、次も同じインスタンスを使えば、キャンシング効果になります。

また、nodejsのモジュールエキスポートは、基本的にキャッシュされます。つまり、ModuleAが複数のファイルに`require`されても、2回目からはキャッシュから使われます。Nodejsのモジュールエキスポートもシングルトンパターンだからです。

```ts
import Kuroshiro from 'kuroshiro'
import KuromojiAnalyzer from 'kuroshiro-analyzer-kuromoji'


class Analyzer {
  private _instance: Kuroshiro
  private isInitialized: boolean

  constructor() {
    this._instance = new Kuroshiro()
    this.isInitialized = false
  }

  private async init() {
    if (!this.isInitialized) {
      await this._instance.init(new KuromojiAnalyzer())
      this.isInitialized = true
    }
    return this._instance
  }

  public async parse(term: string) {
    const parser = await this.init()
    const result = await parser.convert(term.replace(' ', ''), {
      to: 'romaji',
      mode: 'normal',
      romajiSystem: 'nippon'
    })
    return result
  }

  public async getSuggestions(
    tableName: string,
    term: string,
    mode: 'BOOLEAN' | 'NATURAL LANGUAGE' = 'BOOLEAN'
  ) {
    let romaji = await this.parse(term)
    if (!romaji) return [] // もしくはエラーを出すなど

    // 一文字の場合はワイルドガードを追加
    if (romaji.length == 1) {
      romaji += '*'
    }
    const sql = `SELECT name, romaji FROM ${tableName} WHERE MATCH(romaji) AGAINST('${romaji}' IN ${mode} MODE)`
    const results = await db.query(sql) // ここは何かORMなどで実行するのもOK
    return results
  }
}

export default new Analyzer()
```

これでコントローラとかのファイルで：

```ts
import { RequestHandler } from 'express'
import Analyzer from '../path/to/Analyzer'

//...
export const getSuggestions: RequestHandler = async (req, res, next) => {
  try {
    const { tableName, term } = req.query
    if (
      typeof tableName === 'string' &&
      typeof term === 'string' &&
      term.length > 0 &&
      tableName.length > 0
    ) {
      const suggestions = await Analyzer.getSuggestions(tableName, term)
      res.json(suggestions)
    } else {
      throw new Error('tableName or term is missing')
    }
  } catch (error) {
    next(error)
  }
}
```

これで1回目のアクセスは重くなりますが、その後は同じインスタンスを使うので、アナライザーの初期化は一回で済むことになります。

### 最後に

色々と問題がありましたが、なんとか機能するようになりました。

ブールモードの検索には演算子の設定が細かくできるので、使用ケースによって`getSuggestions`の実装を変えたりするのが良いでしょう。

以前PosgreSQLの全文検索機能を使ったことがありますが、MySQLは今回初めてなので、色々と勉強になりました。

ではでは、良きコーディングライフを！
