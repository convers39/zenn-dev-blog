---
title: "Laravel カスタムバリデーションルールを作る"
emoji: "👌"
type: "tech"
topics:
  - "laravel"
  - "validation"
published: true
published_at: "2021-11-23 10:20"
---

リクエストで送られたデータをチェックする時に、FormRequestでバリデーションの処理を行うのが多いでしょう。

Laravelには多くのバリデーションルールが用意されていて、ほとんどの場合は十分ですが、日本語になると、多少対応しきれない部分があります。

例えば、半角文字しか入力が許されないケースがあるとします。これはデフォルトのバリデーションルールにはありませんので、自作で追加します。

```bash
php artisan make:rule HalfWidthChar
```

次のファイルが作られますが、Ruleインターフェースにはpassesとmessageの関数のみとなっていますので、そちらを実装します。

```php
namespace App\Rules;

use Illuminate\Contracts\Validation\Rule;

class HalfWidthChar implements Rule
{
    public function __construct()
    {
        // 必要に応じてencodingを指定
    }

    public function passes($attribute, $value)
    {
	$converted = mb_convert_encoding($value, 'SJIS', 'utf-8');
        return strlen($converted) == mb_strlen($value, 'SJIS');
    }

    public function message()
    {
        return ':attributeは半角文字を入力してください。';
    }
}

```
`strlen`はバイト数をチェックし、`mb_strlen`は文字数をチェックします。全部半角であれば、一文字=1バイトなので、両者が合致するはずです。ただ、ここでエンコーディングの問題があるので、Laravelのデフォルトではutf-8が使われていますが、半角カナを含む半角文字が1バイトのSJISでチェックした方が確実でしょう。エンコーディングを指定したい場合は、コントラクターに入れれば良いですね。

ちなみに、チェックする内容によって、正規表現でパターンでチェックすることできますが、その場合はデフォルトのバリデーションルールのregexを使えば良いですね。例えば、半角英数字のみをチェックしたい場合：

```php
// とあるFormRequestで

 /**
     * Get the validation rules that apply to the request.
     *
     * @return array
     */
    public function rules()
    {
        return [
	    'username' => 'required|regex:/^[a-zA-Z0-9]+$/',
	    'kana' => ['required', new HalfWidthChar],
	    //...
	];   
	
    }
```

もう一つの例として、デフォルトのルールには、`max:10`とかという文字数チェックがあります。またエンコーディングの問題ですが、日本語全角文字の場合、基本的に1文字>=2バイトとなります。しかし、半角も混ざっていたら、このmaxの文字数チェックが無力になります。ここで、DBで使用されるエンコーディングで指定し、文字数ではなく、バイト数で制限するルールを作ります。

```php
namespace App\Rules;

use Illuminate\Contracts\Validation\Rule;
use Illuminate\Support\Facades\Log;

class LengthInBytes implements Rule
{
    private $length;
    private $from_encoding;
    private $to_encoding;

    public function __construct(int $length, string $to_encoding = 'SJIS', string $from_encoding = 'UTF-8')
    {
        $this->length = $length;
        $this->to_encoding = $to_encoding;
        $this->from_encoding = $from_encoding;
    }

    public function passes($attribute, $value)
    {
        return strlen(mb_convert_encoding($value, $this->to_encoding, $this->from_encoding)) <= $this->length;
    }

    public function message()
    {
        return ":attributeを短くしてください。";
    }
}

```

上記の例では、DBではSJISで保存することを想定します。SJISのバイト数が指定のバイト数より超えないことをチェックします。これをFormRequestに導入してみると：

```php
// とあるFormRequestで

 /**
     * Get the validation rules that apply to the request.
     *
     * @return array
     */
    public function rules()
    {
        return [
	    'item_name' => ['required', new LengthInBytes(30)],
	    //...
	];   
	
    }
```

以上です。