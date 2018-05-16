# EOSスマートコントラクト , Part 1: はじめに(Ping equivalent in EOS)

![例](https://github.com/eosasia/ping-eos/raw/master/example.gif)

**Info:** この投稿は、EOSのスマートコントラクト開発の開始から製品化までを手助けする一連のポストの中の最初の投稿です。もしより深く掘り下げたいトピックの提案があれば、気軽にコメントを送ってください。当チュートリアルの全てのコードは[こちら](https://github.com/eosasia/ping-eos)。

全てのEOSの要素の中で、EOSの開発に興味を持っている開発者のほとんどが苦労するのはスマートコントラクトを書き始める部分です。一般的に、新規の開発者にとって次の2つの障壁があります：ツールのセットアップ、そしてスマートコントラクトの記述

EOSのスマートコントラクトはC++で記述され、WebAssemblyにコンパイルされます。[Dan Larimer](https://steemit.com/eos/@dan/eos-example-exchange-contract-and-benefits-of-c)（EOSのCTO）は、C++を使うことでより安全なコントラクトを作ることができ、実行時間が短くなり、ほとんどのメモリに関する懸念が不要になると考えてC++を選択しました。

## セットアップ
EOSを扱う際の課題の1つは、ローカルのブロックチェーンを設定することです。幸運なことに、EOSは[ローカル環境のセットアップのためのガイド](https://github.com/EOSIO/eos/wiki/Local-Environment#getting-the-code) をすでに提供しています。このガイドでは、`EOSIO Dawn 3.0`を使用します。

このガイドの要約は以下のキーとなるコマンドに凝縮できます：
```bash
$ git clone https://github.com/EOSIO/eos --recursive
$ cd eos
$ ./eosio_build.sh
$ cd build && make install
$ cd programs/nodeos
$ ./nodeos -e -p eosio --plugin eosio::wallet_api_plugin --plugin eosio::chain_api_plugin --plugin eosio::account_history_api_plugin --access-control-allow-origin=*
```

インストールには多少の時間がかかります。EOSのブロックチェーンを入手し、ローカルで実行させれば準備完了です。このガイドを通して、`cleos` や `eosiocpp`といった機能を使用しますが、これらは`eos/programs`フォルダに格納されています。

## pingスマートコントラクトをビルドする
このチュートリアルでは、分散型システムping/pongでの“Hello World”を作成し、デプロイします。初めての方のために説明すると、サーバーに“ping”コマンドを送ると、“pong”という返事が帰ってくるシステムです。スマートコントラクト は、C++コード、ABI (Application Binary Interface)、C++コードベースのWAST (Web Assembly Text file)といったいくつかの要素から構成されます。それらは以下のようになっています：
[image:D2EBE9EE-8C93-479F-915D-C69BED670EF5-2033-0000058CE78687BE/Untitled Diagram.png]

### pingの実装
ツールのセットアップが終わったら、コントラクトを書き始めましょう！pingのスマートコントラクトを書くためには、`ping`という一つのアクションを実装するコントラクトだけを書けばよいです。このメソッドに必要なのは、返事に“Pong”を表示することだけです。

`contracts`内に`ping`というフォルダを作成し、その中に `ping/ping.cpp`を以下の内容で作成します：
```c++
#include <eosiolib/eosio.hpp>
#include <eosiolib/print.hpp>

class ping_contract : public eosio::contract {
  public:
      using eosio::contract::contract;
      void ping(account_name receiver) {
         eosio::print("Pong");
      }
};

EOSIO_ABI( ping_contract, (ping) )
```

ここでのアイディアは、動的部分に慣れるためにシンプルで小さな例を試してみることです。ここで何が起こってるか分解してみましょう：
1. コントラクトを書くためのいくつかの定義をインクルードする
2. [eos::contract](https://github.com/EOSIO/eos/blob/8425ff88a7f712c7df46b979de0e6e7de512f569/contracts/eosiolib/contract.hpp)から継承した新しいコントラクトクラスを作成
3. “Pong”を表示するメソッドを作成
4. 最後の行は[最近追加されたマクロ](https://github.com/EOSIO/eos/pull/2051)で、二つめのパラメータに渡したメソッドに基づいたABIを作成することで、私たちの手書きのABIを維持する煩わしさから解放してくれます。

### 自分のコントラクトをビルドしよう
EOSのブロック生成者は、スマートコントラクトを実行するときにC++のコードを実行しません。彼らはweb-assemblyを要求します。EOSは、C++のコードをWebAssemblyテキストに変換するための`eosiocpp` というツールを提供しています。`eosiocpp -o ping.wast ping.cpp`でそれを実行してみましょう。このコンパイルステップはいくつかの警告を表示ますが、今は無視して問題ありません。

次は、Application Binary Interfaceが必要です。基本的に、スマートコントラクト のAPIはメソッドと対応する署名を記述します。私たちは自分でABIを記述する代わりに、`EOSIO_ABI` マクロをファイルの最後に追加しているので、 `eosiocpp -g ping.abi ping.cpp`で簡単に生成することが可能です。

この時点で、あなたのフォルダは以下のようになっているはずです：
```
├── ping.abi
├── ping.cpp
└── ping.wast
```

## ローカルネットへのデプロイ
この時点でスマートコントラクト に必要な要素は揃ったので、デプロイしてみましょう。`cleos wallet create` でウォレットが作成されているか、また、 `cleos wallet unlock`を実行することで解錠されているか確認し、必要であればウォレットのパスワードを入力してください。別のアカウント下でスマートコントラクト をデプロイします。

このため、新しいキーペアを作成することが必要です。`cleos create key`を実行して作成しましょう。これによりランダムなパブリックキーとプライベートキーが生成されます。残りのチュートリアルでは、[public_key]/[private_key]はここで生成された値に置き換えてください。

現在の解錠されているアカウントのウォレットにプライベートキーをインポートする：`cleos wallet import [private_key]`
パブリックキーでアカウントをセットアップする： `cleos create account eosio ping.ctr [owner_key: public_key] [active_key: public_key]`
- 新しく作成されたアカウントとコントラクトをリンクする： `cleos create account eosio ping.ctr [owner_key: public_key] [active_key: public_key]`

## pingとやりとりする
新しいコントラクトがデプロイされたら、それとやりとりしてみましょう！トランザクションを実行するために同じキーペアでテストアカウントを作りましょう：`cleos create account eosio tester [public_key] [public_key]`

コマンドラインでテストすることができます：
```bash
$ cleos push action ping.ctr ping '["tester"]' -p tester
executed transaction: e89ebeaad8f3623e42d966f62d4d0adbf6a7412b6bb4d7be61f04a22d3cd485e  232 bytes  102400 cycles
#  ping.ctr <= ping.ctr::ping           {"account":"tester"}
>> Received ping
```

動きましたね！

私たちのようなプログラマーにとってはこれでもいいですが、あなたのスマートコントラクト のユーザーたちはわざわざコマンドラインでセットアップしたくはないでしょう。そのため、このやりとりをより身近なインターフェースであるブラウザに移行しましょう。

### ブラウザを通してやりとりする
フロントエンドからEOSとやりとりするためには、[EOS.js](https://github.com/EOSIO/eosjs)を使用します。私たちはdawn3を使っているので、`npm install eosjs@dawn3`をインストールする際には、`dawn3`ブランチを使っているか確認する必要があります。

以下の設定から始めます：
```javascript
Eos = require('eosjs')

eos = EOS.Localnet({
  keyProvider: ['{replace_with_your_private_key}'],
  httpEndpoint: 'http://127.0.0.1:8888'
})
```

設定できたら、いくつか詳細を明記する必要があります：
```javascript
eos.contract('ping.ctr').then((contract) => {
  contract.ping("tester", { authorization: ['tester'] }).then((res) => {
    console.log(res)
  })
})
```

`ping.ctr`が前にデプロイしたコントラクトの名前とマッチすることに注目してください。一度コントラクトのインターフェース（またはABI）をフェッチしたら、あたかもそれがネイティブのJavascript であるかのようにやりとりすることができます。コントラクトは、resolve関数にインクルードされたトランザクションの詳細とpromiseを返します。これがpingスマートコントラクトをフロントエンドからやりとりするコアアイディアです。

動作例と一緒により大きな文脈からこれを確認するためには、[このレポジトリ](https://github.com/eosasia/ping-eos/tree/master/frontend)でフロントエンドコードを確認してみください。


Follow us on: 
[Twitter](https://twitter.com/EOSAsia_one)
[Medium](https://medium.com/@eosasia)
[Website](https://www.eosasia.one/)
[Github](https://github.com/eosasia)
