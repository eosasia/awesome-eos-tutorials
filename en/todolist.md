# Part 2: Building a To-do list with EOS (or Working with persistent data in EOS)
[gif]

[info] This post is the second in a series of posts made to help EOS smart contracts developers go from beginner-to-production. If you have any suggestions of topics you’d like to a deep-dive on, please leave it in the comments! If you’re not familiar with deploying a simple smart contract yet, I suggest you check out [the previous tutorial](https://steemit.com/devs/@eos-asia/eos-smart-contracts-part-1-getting-started-ping-equivalent-in-eos).

The majority of smart contract use-cases require developers to work with persistent data on the blockchain. In this tutorial, we’ll go through an example scenario where we build a todo list and teach you how to implement the standard operations (CRUD) when working with data.

## Diving into Boost.MultiIndex
Since EOS smart contracts are based on C++, we make use of a library called [Boost.MultiIndex Containers](https://www.boost.org/doc/libs/1_63_0/libs/multi_index/doc/index.html). In its own words:

> The Boost Multi-index Containers Library provides a class template named multi_index_container which enables the construction of containers maintaining one or more indices with different sorting and access semantics. Indices provide interfaces similar to those of STL containers, making using them familiar. The concept of multi-indexing over the same collection of elements is borrowed from relational database terminology and allows for the specification of complex data structures in the spirit of multiply indexed relational tables where simple sets and maps are not enough.  

Let’s break that description down by explaining some concepts and associating them against something developers with traditional database experience would be more familiar with:
- Containers: Class that contains many elements (table)
- Elements: Data objects (rows in a table)
- Interface: Container methods that retrieve elements (query)

In EOS smart contracts, you use multi-index containers by defining them with `eosio::multi_index`. If you look at example contracts that make use of this feature, like the [dice contract example](https://github.com/EOSIO/eos/blob/master/contracts/dice/dice.cpp), it can be a little daunting to figure out exactly which piece is dealing with persistent data manipulation. (Nothing to fear, we’ll help you understand this and soon enough you can implement smart-contracts with storage on your own.)

The narrative throughout our explanation will be to build a todo list. We should be able to check off items we’ve done, add new items, and remove items that are no longer needed. Given this example, we’ll be using `todos` for our container name, and `todo` for our element structure.

We’ll get started with initializing our first container, first we pass two template parameters into `eosio::multi_index`. The first will be the name of our container, and the second is the data structure defining the element. Let’s built a very minimal example of our todo model:
```c++
struct todo {
  uint64_t id;

  uint64_t primary_key() const { return id; }
  EOSLIB_SERIALIZE(todo, (id))
};

typedef eosio::multi_index<N(todos), todo> todo_table;
todo_table todos;
```

This works! We simply define an ID as 64bit (unsigned) integer, and create a way to access it (via primary_key) We define our multi_index as a typedef since we don’t want to instantiate it yet. At this point, our todo model it isn’t quite useful yet. Let’s add a few additional properties:
```c++
struct todo {
  uint64_t id;
  std::string description;
  uint64_t completed;

  EOSLIB_SERIALIZE(todo, (id)(description)(completed))
};

typedef eosio::multi_index<N(todos), todo> todo_table;
todo_table todos;
```

Now we’re getting there. A description of the todo (e.g. “Finish writing novel”) and a state to determine whether it’s been completed or not should be enough for now.

For the convenience of auto-generating our ABI (Application Binary Interface), we’ll specify a type comment that helps the generator above the container definition: `@abi table profiles i64`.

Incase you’re wondering what that `i64` in the comment above means. This is our lookup index. By default, we need a way to lookup elements in our container, so our first 64 bits (basically the first key if it’s a 64bit type) will serve that purpose. It’s common to use `uint64_t id;`  for the first key, but you could also use an `account_name` type [since account_name is also uint64_t under the hood](https://github.com/EOSIO/eos/blob/2f2c8c7e3811caca178a7553192c8fe59a22576d/contracts/eosiolib/types.h#L22).

At this point, we should have a minimally functioning container who’s code should look something like:
```c++
// @abi table todos i64
struct todo {
  uint64_t id;
  std::string description;
  uint64_t completed;

  uint64_t primary_key() const { return id; }
  EOSLIB_SERIALIZE(todo, (id)(description)(completed))
};

typedef eosio::multi_index<N(todos), todo> todo_table;
todo_table todos;
```

## Working with your new container
Now that we have a container defined, we can work with the elements inside of it. The way we’ll handle mutating those elements will be via actions in our smart contract.

There are four basic functions of persistent storage: Create, Retrieve, Update, and Delete. In our case, we don’t have to worry about Retrieve, since that will be handled by the front-end loading our contract instead of an action. For the other three, we’ll create an action for each:

### Create
Appending a todo item to our todo list can be done with `emplace`:
```c++
// @abi action
void create(account_name author, const uint32_t id, const std::string& description) {
  todos.emplace(author, [&](auto& new_todo) {
    new_todo.id  = id;
    new_todo.description = description;
    new_todo.completed = 0;
  });

  eosio::print("todo#", id, " created");
}
```
An important detail to not here is that we pass in the author as a parameter as well. This is necessary for the first parameter in the `emplace` method.

### Update/Complete 
We’ll create an action to complete our todo by updating the `completed` attribute. Like so:
```c++
// @abi action
void complete(account_name author, const uint32_t id) {
  auto todo_lookup = todos.find(id);
  eosio_assert(todo_lookup != todos.end(), "Todo does not exist");

  todos.modify(todo_lookup, author, [&](auto& modifiable_todo) {
    modifiable_todo.completed = 1;
  });

  eosio::print("todo#", id, " marked as complete");
}
```

### Delete
Given this is an internal smart contract, we don’t have to worry about security or permissions quite yet. This allows us to focus on the minimal working version of the delete action:
```c++
// @abi action
void destroy(account_name author, const uint32_t id) {
  auto todo_lookup = todos.find(id);
  todos.erase(todo_lookup);

  eosio::print("todo#", id, " destroyed");
}
```

## Deploying, testing, and hooking actions up to a front-end
Our previous tutorial linked an EOS smart contract with a webpage front-end using a simple ping/pong example. Now that we have our actions to work with persistent data on the EOS blockchain, we can build the front end for our todo list.

### Deploying
Deploying our contract should be straightforward, let’s step through it:
- Build the contract ABI and WASM: `eosiocpp -o hello.wast hello.cpp && eosiocpp -g hello.abi hello.cpp`
- Set up account/wallet: 
```bash
cleos create account eosio todo.user EOS7ijWCBmoXBi3CgtK7DJxentZZeTkeUnaSDvyro9dq7Sd1C3dC4 EOS7ijWCBmoXBi3CgtK7DJxentZZeTkeUnaSDvyro9dq7Sd1C3dC4

cleos set contract todo.user ../todo -p todo.user
```

Testing the contract is straightforward:
```bash
$ cleos push action todo create '["todo", 1, "hello world"]' -p todo.user
executed transaction: bc5bfbd1e07f6e3361d894c26d4822edcdc2e42420bdd38b46a4fe55538affcf  248 bytes  107520 cycles
#          todo <= todo::create                 {"author":"todo","id":1,"description":"hello world"}
>> todo created
```

Then accessing the data can be done like so:
```bash
$ cleos get table todo todo todos
```

### Testing on the front-end
I’ll spare the readers from delving into the React.js code, though I highly encourage you check it out for yourselves in [the example repository](https://github.com/eosasia/eos-todo). 

If you folks would like us to dive deeper on what it takes to hook up your EOS contracts with a browser-based front end, whether using React, Vue, Angular, or plain ol’ Javascript, let us know in the comments below!
