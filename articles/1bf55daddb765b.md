---
title: "WindowsでLaravelのFTPサーバーを使う"
emoji: "🕌"
type: "tech"
topics: []
published: false
---

Laravelのfilestorageシステムでは、ローカルだけでなく、AWS S3などクラウドサービスも統合されています。適切に設定すれば、3rd partyのパッケージに頼らず使うことが可能です。

今回はデフォルトには書かれていないが、時に需要があるFTPサーバーの設定について紹介します。

#### WindowsでFTPサーバーのセットアップ

セットアップ⼿順は主に[こちら](https://winscp.net/eng/docs/guide_windows_ftps_server)の記事と[こちら](https://www.youtube.com/watch?v=Cd2k9nQSdro)のビデオに参考しています。

まずはFTP とIIS インターネットインフォメーションサービス管理コンソールをオンにします。
![](https://storage.googleapis.com/zenn-user-upload/5616d5f1de1a23f766ed7d01.png)

これでインストールが完了したら、スタートメニューからIIS を検索して、IIS マネジャーを開き ます。
![](https://storage.googleapis.com/zenn-user-upload/5f6e1b5460aef2f4beb409f1.png)

FTP のサイトを追加します。
![](https://storage.googleapis.com/zenn-user-upload/1503572ffcb3e78dae0fd9a3.png)

次にファイアウォールの設定で、FTP ⽤の通信規則（受信・送信両⽅）を有効にします。
![](https://storage.googleapis.com/zenn-user-upload/f5476c29c3389b1d25b288fe.png)

設定変更を適応するために、FTP サービスを再起動します。
![](https://storage.googleapis.com/zenn-user-upload/d59444c60b32672c8d6d2e66.png)

これでPowerShellでftp接続してみると、現在windowsのユーザー名とパスワードでログインできます。
![](https://storage.googleapis.com/zenn-user-upload/cde04e71d330b465004a8b40.png)

もちろん、FTP専用のユーザーまたはユーザーグループを作っても大丈夫ですが、もしログインできない場合はこちらの承認規則で追加すれば良い。
![](https://storage.googleapis.com/zenn-user-upload/e460a5fb6890937308e671bc.png)

筆者が経験した一つのトラップですが、FileZilla ServerとのFTPサービス（FileZillaのFTPクライエントとは別）を利用する場合、ログインできない問題がありました。

![](https://storage.googleapis.com/zenn-user-upload/212d1f15db2f72c381364832.png)

理由は不明だが、サービスの中でFileZilla Serverとのサービスを無効にすれば、デフォルトのMicrosoft FTP Serviceに戻ります。それで無事にログインできます。

#### LaravelでFTPディスクを設定

Laravelの`config/filesystems.php`にFTPディスクを追加します。ホスト名はホストマシンのipアドレス（ローカルの場合はlocalhostでも可能）、ユーザー名とパスワードは先ほど設定したftpサーバーのユーザーになると。
```php
    'disks' => [
	// ...
        'ftp' => [
            'driver' => 'ftp',
            'host' => env('FTP_HOST'),
	    // 'port' => 21,
            'username' => env('FTP_USERNAME'),
            'password' => env('FTP_PASSWORD'),
            'root' => '',
            // 'root' => env('FTP_ROOT_DIR'),
            'passive' => true,
            'ignorePassiveAddress' => true,
            // 'ssl' => true,
            // 'timeout' => 30,
	    // 'permPrivate',
            // 'permPublic',
            // 'transferMode',
            // 'systemType',
            // 'recurseManually',
            // 'utf8',
            // 'enableTimestampsOnUnixListings',
        ]
```

一部の手引きではdriverからrootまでしか書いていなく、これでもしパッシブ接続モードの設定をしていなければ、機能出来ない可能性がありますため、ここはignorePassiveAddressをtrueに変更します。

これで設定が終わって、一旦tinkerでファイルの読み込みと転送を試してみる：
```shell
Storage::disk('ftp')->allFiles();
$file = Storage::disk('local')->get('test.txt');
$path = 'ftp_folder/ftptest.txt'
Storage::disk('ftp')->put($path, $file);
```

もしtrueがリターンされたら転送が成功です。これでコントローラとかで保存するディスクをftpに指定すればファイルはそちらに保存されます。

#### FTP接続のモードについて
設定する中でよく分からなかったのは、passive FTP との⾔葉でした。passive があれば、active  もあるでしょう。そこでFTP の接続モードについて調べてみました。

まずFTP の接続には、command/control とdata との2 つの通信チャンネルがあります。コマンド チャンネルは通信を開始するために、ポート情報を送ったり、接続したりするチャンネルです が、実際のファイル送信はデータチャンネルで実⾏されます。

Active モードの流れとして：

- クライエント側がコマンドチャンテルを立ち上げ、PORT コマンドで⾃分のIP アドレスと ランダムなポート（データチャンネル⽤）をサーバーのポート21へ送る
- サーバー側がポート20 からデータチャンネルを立ち上げ、指定されたアドレスとポートへ 接続を始める
- 接続が確立できると、ファイルの転送を開始する

Passive モードの流れとして： 
- クライエント側がランダムなポートから、コマンドチャンテルを立ち上げ、PASV コマン ドでサーバーポート21 にどのIPアドレスとポートを使えば良いのかを聞く
- サーバー側が接続⽤のIPアドレスとポート（ランダム）を返信する
- クライエント側がランダムなポートからデータチャンネルを立ち上げ、指定されたアドレ スとポートへ接続する
- 接続が確立できると、ファイルの転送を開始する

図で表現すると：
![](https://storage.googleapis.com/zenn-user-upload/98d41ecf5d22a17678dd46d5.png)
|  Active  |  Passive  |
| ---- | ---- |
|  クライエント側がコマンドチャンネル、サーバーがデータチャンネルを立ち上げる  |  コマンドもデータも両方クライエント側により立ち上げられる  |
|  FTPサーバーに安全保証  |  安全面保証しない  |
|  ファイアウォールにより問題が起こる  |  ファイアウォール問題の心配はない  |
|  デフォルトモードではない  |  デフォルトモード  |

なぜこのモードの区別が重要かというと、ファイアウォールの通信制限と関わるからのようです。アクティブモードでは、データチャンネルがサーバー側に立ち上げられ、サーバー側がクライエント側へ接続しようとする流れになっています。クライエント側では通常ファイアウォール・NATの設定があるため、ランダムなポートへのインバウンド通信がブロックされるかもしれません。それに対して、パッシブモードのデータチャンネルが、クライエント側に立ち上げられるため、アウトバウンド通信は通常ブロックされるリスクが少ないと。そのため、パッシブモードでのFTP設定も重要になってくると思われます。

パッシブモードで懸念すべきところとして、安全⾯の保証らしい。サーバー側がランダムなポートを開放するため、リスクを軽減するために、使⽤するポートの区間・レンジを指定することが勧められています。
![](https://storage.googleapis.com/zenn-user-upload/9547f467b3feb2bc72ce97d2.png)


以上！


