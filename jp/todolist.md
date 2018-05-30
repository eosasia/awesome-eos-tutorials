# Part 2: EOSを使ってTo-doリストを作る(EOSで永続データを扱う）
[gif]

[info]この投稿は、EOSのスマートコントラクト開発の開始から製品化までを手助けする一連のポストの中の2番目の投稿です。 より深く掘り下げたいトピックの提案があれば、コメントでお知らせください！シンプルなスマートコントラクトの開発について学びたい場合は[前のチュートリアル](https://steemit.com/devs/@eos-asia/eos-smart-contracts-part-1-getting-started-ping-equivalent-in-eos)をご参照ください。

スマートコントラクトを使用する場合、基本的に開発者は永続データをブロックチェーン上で操作する必要があります。 このチュートリアルでは、ToDoリストを作成し、データ操作の際に標準操作(CRUD)を実装する方法を説明します。

## Diving into Boost.MultiIndex
EOSのスマートコントラクトはC++に基づいているため、[Boost.MultiIndex Containers](https://www.boost.org/doc/libs/1_63_0/libs/multi_index/doc/index.html)というライブラリを利用しています。Boost.MultiIndex Containersに関する本家の説明は以下の通りです：

> Boostマルチインデックスコンテナライブラリには、異なるソートおよびアクセスセマンティクスを持つ1つ以上のインデックスを保持するコンテナの構築を可能にする multi_index_container という名前のクラステンプレートが用意されています。インデックスは、STLコンテナと同様のインターフェースを提供し、使い慣れたものを使用します。同じ要素の集合に対するマルチ・インデックスの概念は、リレーショナル・データベース用語から借用され、単純なセットとマップでは不十分な多重索引リレーショナル・テーブルの中で複雑なデータ構造の仕様を可能にします。

いくつかのコンセプトを説明し、従来型のデータベースの経験を持つ開発者にとってより馴染みの深いものと関連付けることで、より具体的に説明してみましょう：
- コンテナ:多くの要素を含むクラス(テーブル)
- 要素:データオブジェクト(テーブル内の行)
- インタフェース:要素を検索するコンテナメソッド(クエリ)

EOSのスマートコントラクトでは、マルチインデックスコンテナを `eosio::multi_index`で定義して使用します。[ダイスコントラクトの例](https://github.com/EOSIO/eos/blob/master/contracts/dice/dice.cpp)のように、この機能を利用するコントラクトの例を見てみると、どの部分が永続データの操作を処理しているか正確に把握するのは少し困難です。(恐れることはありません、私達が理解の手助けをするので、すぐに自分でストレージを使ってスマートコントラクトを実装できるようになります)。

この説明を通して実現することは、ToDoリストを構築することです。完了したアイテムをチェックしたり、新しいアイテムを追加したり、不要になったアイテムを削除したりすることができます。この例では、コンテナ名に`ToDos`、要素構造に`ToDo`を使用します。

それでは、最初のコンテナの初期化から初めましょう。まず、2つのテンプレートパラメータを`eosio::multi_index`に渡します。最初はコンテナの名前、2番目は要素を定義するデータ構造です。ToDoモデルの最小限の例を作ってみましょう:
```c++
struct ToDo {
  uint64_t id;

  uint64_t primary_key() const { return id; }
  EOSLIB_SERIALIZE(ToDo, (id))
};

typedef eosio::multi_index<N(ToDos), ToDo> ToDo_table;
ToDo_table ToDos;
```

動きましたね！IDを64ビット(符号なし)の整数として定義し、それにアクセスする方法を作成します(primary_key経由)。マルチインデックスは、まだインスタンス化したくないので、typedefとして定義します。この時点で、私たちのToDoモデルはそれほど有用ではありません。いくつかプロパティを追加しましょう:
```c++
struct ToDo {
  uint64_t id;
  std::string description;
  uint64_t completed;

  EOSLIB_SERIALIZE(ToDo, (id)(description)(completed))
};

typedef eosio::multi_index<N(ToDos), ToDo> ToDo_table;
ToDo_table ToDos;
```

順調です。ToDoの記述（e.g. "小説の執筆を完了する")とそれが完了されたか決定するためのステートは今のところ十分なはずです。

ABI(Application Binary Interface)の自動生成の便宜のため、コンテナ定義上のジェネレータを助けるタイプコメントを指定します: `@abi table profiles i64`。

上記の`i64`が何を意味しているのか疑問に思うかもしれません。これはルックアップインデックスです。デフォルトでは、コンテナ内の要素を参照する方法が必要なので、最初の64ビット(基本的に64ビットの場合は最初のキー)がその役割を果たします。最初のキーには`uint64_t id;`を使うのが一般的ですが、[account_nameもuint64_tなので](https://github.com/EOSIO/eos/blob/2f2c8c7e3811caca178a7553192c8fe59a22576d/contracts/eosiolib/types.h#L22)`account_name`タイプを使うこともできます。

この時点で、最小限の機能を持つコンテナを用意する必要があります。このコンテナのコードは次のようになります。
```c++
// @abi table ToDos i64
struct ToDo {
  uint64_t id;
  std::string description;
  uint64_t completed;

  uint64_t primary_key() const { return id; }
  EOSLIB_SERIALIZE(ToDo, (id)(description)(completed))
};

typedef eosio::multi_index<N(ToDos), ToDo> ToDo_table;
ToDo_table ToDos;
```

## 新しいコンテナを使用する
コンテナが定義されたので、内部の要素を操作できます。スマートコントラクト内のアクションを経由することで、これらの要素を変化させることができます。

永続ストレージには、Create、Retrieve、Update、およびDeleteの4つの基本機能があります。今回のケースでは、フロントエンドがアクションの代わりにコントラクトを読み込むことで処理されるため、Retrieveについて心配する必要はありません。 他の3つについては、それぞれのアクションを作成します。

### Create
ToDoアイテムをToDoリストに追加するには、`emplace`を実行します:
```c++
// @abi action
void create(account_name author, const uint32_t id, const std::string& description) {
  ToDos.emplace(author, [&](auto& new_ToDo) {
    new_ToDo.id  = id;
    new_ToDo.description = description;
    new_ToDo.completed = 0;
  });

  eosio::print("ToDo#", id, " created");
}
```
パラメータとしてauthorを渡すということはここでは重要ではありません。 これは、`emplace`メソッドの最初のパラメータに必要です。

### Update/Complete
`completed`属性を更新することで、ToDoを完了するアクションを作成します:
```c++
// @abi action
void complete(account_name author, const uint32_t id) {
  auto ToDo_lookup = ToDos.find(id);
  eosio_assert(ToDo_lookup != ToDos.end(), "ToDo does not exist");

  ToDos.modify(ToDo_lookup, author, [&](auto& modifiable_ToDo) {
    modifiable_ToDo.completed = 1;
  });

  eosio::print("ToDo#", id, " marked as complete");
}
```

### Delete
これは内部のスマートコントラクトであるため、セキュリティやアクセス許可についてまだ心配する必要はありません。これにより、deleteアクションの最小限のワーキングバージョンに焦点を当てることができます。
```c++
// @abi action
void destroy(account_name author, const uint32_t id) {
  auto ToDo_lookup = ToDos.find(id);
  ToDos.erase(ToDo_lookup);

  eosio::print("ToDo#", id, " destroyed");
}
```

## フロントエンドまでのアクションのデプロイ、テスト、フック
前回のチュートリアルでは、単純なping/pongの例を使ってEOSのスマートコントラクトをウェブページのフロントエンドにリンクしました。これでEOSブロックチェーンの永続的なデータを扱えるようになったので、ToDoリストのフロントエンドを構築することができます。

### デプロイ
コントラクトのデプロイは難しくないはずなので、やってみましょう:
- コントラクトABIとWASMを構築する: `eosiocpp -o hello.wast hello.cpp && eosiocpp -g hello.abi hello.cpp`
- コントラクトのABIとWASMを構築する: `eosiocpp -o hello.wast hello.cpp && eosiocpp -g hello.abi hello.cpp`
- アカウント/ウォレットを設定する:

```bash
cleos create account eosio ToDo.user EOS7ijWCBmoXBi3CgtK7DJxentZZeTkeUnaSDvyro9dq7Sd1C3dC4 EOS7ijWCBmoXBi3CgtK7DJxentZZeTkeUnaSDvyro9dq7Sd1C3dC4

cleos set contract ToDo.user ../ToDo -p ToDo.user
```

コントラクトのテストも簡単です:
```bash
$ cleos push action ToDo create '["ToDo", 1, "hello world"]' -p ToDo.user
executed transaction: bc5bfbd1e07f6e3361d894c26d4822edcdc2e42420bdd38b46a4fe55538affcf  248 bytes  107520 cycles
#          ToDo <= ToDo::create                 {"author":"ToDo","id":1,"description":"hello world"}
>> ToDo created
```

これで以下のようにデータにアクセスできます:
```bash
$ cleos get table ToDo ToDo ToDos
```

### フロントエンドでのテスト
読者がReact.jsコードを掘り下げないようにしますが、[サンプルリポジトリ](https://github.com/eosasia/eos-ToDo)で確認しておくことを強くお勧めします。

React、Vue、Angular、Javascriptを使用しているかどうかにかかわらず、ブラウザベースのフロントエンドとEOSコントラクトを結びつけるために必要なことについてより深く知りたい場合は、下記のコメントでお知らせください！
