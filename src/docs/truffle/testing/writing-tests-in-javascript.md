---
title: Writing Tests in JavaScript
layout: docs.hbs
---
# Writing Tests in JavaScript

Truffle uses the [Mocha](https://mochajs.org/) testing framework and [Chai](http://chaijs.com/) for assertions to provide you with a solid framework from which to write your JavaScript tests. Let's dive in and see how Truffle builds on top of Mocha to make testing your contracts a breeze.

Note: If you're unfamiliar with writing unit tests in Mocha, please see [Mocha's documentation](https://mochajs.org/) before continuing.

## Use contract() instead of describe()

Structurally, your tests should remain largely unchanged from that of Mocha: Your tests should exist in the `./test` directory, they should end with a `.js` extension, and they should contain code that Mocha will recognize as an automated test. What makes Truffle tests different from that of Mocha is the `contract()` function: This function works exactly like `describe()` except it enables Truffle's [clean-room features](/docs/truffle/testing/testing-your-contracts#clean-room-environment). It works like this:

* Before each `contract()` function is run, your contracts are redeployed to the running Ethereum client so the tests within it run with a clean contract state.
* The `contract()` function provides a list of accounts made available by your Ethereum client which you can use to write tests.

Since Truffle uses Mocha under the hood, you can still use `describe()` to run normal Mocha tests whenever Truffle clean-room features are unnecessary.

## Use contract abstractions within your tests

Contract abstractions are the basis for making contract interaction possible from JavaScript (they're basically our [flux capacitor](https://www.youtube.com/watch?v=EhU862ONFys)). Because Truffle has no way of detecting which contracts you'll need to interact with within your tests, you'll need to ask for those contracts explicitly. You do this by using the `artifacts.require()` method, a method provided by Truffle that allows you to request a usable contract abstraction for a specific Solidity contract. As you'll see in the example below, you can then use this abstraction to make sure your contracts are working properly.

For more information on using contract abstractions, see the [Interacting With Your Contracts](/docs/truffle/getting-started/interacting-with-your-contracts) section.

## Using artifacts.require()

Using `artifacts.require()` within your tests works the same way as using it within your migrations; you just need to pass the name of the contract. See the [artifacts.require() documentation](/docs/truffle/getting-started/running-migrations#artifacts-require-) in the Migrations section for detailed usage.

## Using web3

A `web3` instance is available in each test file, configured to the correct provider. So calling `web3.eth.getBalance` just works!

## Examples

### Using `.then`

Here's an example test provided in the [MetaCoin Truffle Box](/boxes/metacoin). Note the use of the `contract()` function, the `accounts` array for specifying available Ethereum accounts, and our use of `artifacts.require()` for interacting directly with our contracts.

File: `./test/metacoin.js`

```javascript
const MetaCoin = artifacts.require("MetaCoin");

contract("MetaCoin", accounts => {
  it("should put 10000 MetaCoin in the first account", () =>
    MetaCoin.deployed()
      .then(instance => instance.getBalance.call(accounts[0]))
      .then(balance => {
        assert.equal(
          balance.valueOf(),
          10000,
          "10000 wasn't in the first account"
        );
      }));

  it("should call a function that depends on a linked library", () => {
    let meta;
    let metaCoinBalance;
    let metaCoinEthBalance;

    return MetaCoin.deployed()
      .then(instance => {
        meta = instance;
        return meta.getBalance.call(accounts[0]);
      })
      .then(outCoinBalance => {
        metaCoinBalance = outCoinBalance.toNumber();
        return meta.getBalanceInEth.call(accounts[0]);
      })
      .then(outCoinBalanceEth => {
        metaCoinEthBalance = outCoinBalanceEth.toNumber();
      })
      .then(() => {
        assert.equal(
          metaCoinEthBalance,
          2 * metaCoinBalance,
          "Library function returned unexpected function, linkage may be broken"
        );
      });
  });

  it("should send coin correctly", () => {
    let meta;

    // Get initial balances of first and second account.
    const account_one = accounts[0];
    const account_two = accounts[1];

    let account_one_starting_balance;
    let account_two_starting_balance;
    let account_one_ending_balance;
    let account_two_ending_balance;

    const amount = 10;

    return MetaCoin.deployed()
      .then(instance => {
        meta = instance;
        return meta.getBalance.call(account_one);
      })
      .then(balance => {
        account_one_starting_balance = balance.toNumber();
        return meta.getBalance.call(account_two);
      })
      .then(balance => {
        account_two_starting_balance = balance.toNumber();
        return meta.sendCoin(account_two, amount, { from: account_one });
      })
      .then(() => meta.getBalance.call(account_one))
      .then(balance => {
        account_one_ending_balance = balance.toNumber();
        return meta.getBalance.call(account_two);
      })
      .then(balance => {
        account_two_ending_balance = balance.toNumber();

        assert.equal(
          account_one_ending_balance,
          account_one_starting_balance - amount,
          "Amount wasn't correctly taken from the sender"
        );
        assert.equal(
          account_two_ending_balance,
          account_two_starting_balance + amount,
          "Amount wasn't correctly sent to the receiver"
        );
      });
  });
});
```


This test will produce the following output:

```
  Contract: MetaCoin
    √ should put 10000 MetaCoin in the first account (83ms)
    √ should call a function that depends on a linked library (43ms)
    √ should send coin correctly (122ms)


  3 passing (293ms)
```

### Using async/await

Here is a similar example, but using [async/await](https://javascript.info/async-await) notation:

```javascript
const MetaCoin = artifacts.require("MetaCoin");

contract("2nd MetaCoin test", async accounts => {
  it("should put 10000 MetaCoin in the first account", async () => {
    const instance = await MetaCoin.deployed();
    const balance = await instance.getBalance.call(accounts[0]);
    assert.equal(balance.valueOf(), 10000);
  });

  it("should call a function that depends on a linked library", async () => {
    const meta = await MetaCoin.deployed();
    const outCoinBalance = await meta.getBalance.call(accounts[0]);
    const metaCoinBalance = outCoinBalance.toNumber();
    const outCoinBalanceEth = await meta.getBalanceInEth.call(accounts[0]);
    const metaCoinEthBalance = outCoinBalanceEth.toNumber();
    assert.equal(metaCoinEthBalance, 2 * metaCoinBalance);
  });

  it("should send coin correctly", async () => {
    // Get initial balances of first and second account.
    const account_one = accounts[0];
    const account_two = accounts[1];

    const amount = 10;

    const instance = await MetaCoin.deployed();
    const meta = instance;

    const balance = await meta.getBalance.call(account_one);
    const account_one_starting_balance = balance.toNumber();

    balance = await meta.getBalance.call(account_two);
    const account_two_starting_balance = balance.toNumber();
    await meta.sendCoin(account_two, amount, { from: account_one });

    balance = await meta.getBalance.call(account_one);
    const account_one_ending_balance = balance.toNumber();

    balance = await meta.getBalance.call(account_two);
    const account_two_ending_balance = balance.toNumber();

    assert.equal(
      account_one_ending_balance,
      account_one_starting_balance - amount,
      "Amount wasn't correctly taken from the sender"
    );
    assert.equal(
      account_two_ending_balance,
      account_two_starting_balance + amount,
      "Amount wasn't correctly sent to the receiver"
    );
  });
});
```

This test will produce identical output to the previous example.

## Specifying tests

You can limit the tests being executed to a specific file as follows:

```shell
truffle test ./test/metacoin.js
```

See the full [command reference](/docs/truffle/reference/truffle-commands#test) for more information.

## Advanced

Truffle gives you access to Mocha's configuration so you can change how Mocha behaves. See the [project configuration](/docs/advanced/configuration#mocha) section for more details.

## TypeScript File Support

Truffle supports tests saved as a `.ts` [TypeScript](https://www.typescriptlang.org/) file. Please see the [Writing Tests in JavaScript](#writing-tests-in-javascript) guide for more information.
