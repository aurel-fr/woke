# Getting started

This guide explains how to run the first test in Woke development and testing framework.

!!! warning "Important"
    Before getting started, make sure to have the latest version of a development chain installed.

    This is especially important in the case of [Anvil](https://github.com/foundry-rs/foundry/tree/master/anvil), because it is under active development.
    To install the latest version of Anvil, run the following command:

    ```shell
    foundryup
    ```

## Generating pytypes

`pytypes` are Python-native equivalents of Solidity types. They are generated from Solidity source code and used in tests and deployment scripts to interact with smart contracts.

The first step is to generate `pytypes` by running the following command:

```shell
woke init pytypes -w
```

!!! note "Configuring compilation"
    Woke uses default configuration options that should work for most projects.
    However, in some cases, it may be necessary to configure the compilation process.
    For more information, see the [Compilation](../compilation.md) page.

This command creates a `pytypes` directory in the current working directory. The `-w` flag tells Woke to watch for changes in the smart contracts and automatically regenerate `pytypes` when a change is detected.

<div id="generating-pytypes-asciinema" style="z-index: 1; position: relative;"></div>
<script>
  window.onload = function(){
    AsciinemaPlayer.create('../generating-pytypes.cast', document.getElementById('generating-pytypes-asciinema'), { preload: true, autoPlay: true, rows: 15 });
}
</script>

When a compilation error occurs, Woke generates `pytypes` for the contracts that were successfully compiled. `pytypes` for the contracts that failed to compile are not generated.

!!! warning "Name collisions in `pytypes`"
    In some cases, a name of a Solidity types may be a keyword in Python or otherwise reserved name. In such cases, Woke will append an underscore to the name of the type. For example, `class` will be renamed to `class_`.

    This also applies to overloaded functions. For example, if a contract has a function `foo` that takes an argument of type `uint256` and another function `foo` that takes an argument of type `uint8`, the generated `pytypes` will contain two functions `foo` and `foo_`.

## Writing the first test

!!! tip
    Solidity source code for all examples in this guide is available in the [Woke repository](https://github.com/Ackee-Blockchain/woke/tree/main/examples/testing).

To collect and execute tests, Woke uses the [pytest](https://docs.pytest.org/en/stable/) framework under the hood.
The test files should start with `test_` or end with `_test.py` to be collected. It is possible to use all the features of the pytest framework like [fixtures](https://docs.pytest.org/en/stable/explanation/fixtures.html).

The recommended project structure is as follows:

```text
.
├── contracts
│   └── Counter.sol
├── pytypes
├── scripts
│   ├── __init__.py
│   └── deploy.py
└── tests
    ├── __init__.py
    └── test_counter.py
```

### Connecting to a chain

In single-chain tests, it is recommended to use the `default_chain` object that is automatically created by Woke.
The `connect` decorator either launches a new development chain or connects to an existing one, if the second argument is specified.
It is possible to connect using:

- a HTTP connection (e.g. `http://localhost:8545`),
- a WebSocket connection (e.g. `ws://localhost:8545`),
- an IPC socket (e.g. `/tmp/anvil.ipc`).

```python
from woke.testing import *

# launch a new development chain
@default_chain.connect()
# or connect to an existing chain
# @default_chain.connect("ws://localhost:8545")
def test_counter():
    print(default_chain.chain_id)
```

To run the test, execute the following command:

```shell
woke test tests/test_counter.py -d
```

The `-d` flag tells Woke to attach the Python debugger on test failures.

### Deploying a contract

Every Solidity source file has its equivalent in the `pytypes` directory. These directories form a module hierarchy that is similar to the one in the `contracts` directory.
The `Counter` contract from the previous example is available in the `pytypes.contracts.Counter` module.

Every contract has a `deploy` method that deploys the contract to the chain.
The `deploy` method accepts the arguments that are required by the contract's constructor.
Additionally, it accepts keyword arguments that can be used to configure the transaction that deploys the contract.
All keyword arguments are described in the [Interacting with contracts](./interacting-with-contracts.md) section.

```python
from woke.testing import *

from pytypes.contracts.Counter import Counter

@default_chain.connect()
def test_example():
    counter = Counter.deploy(from_=default_chain.accounts[0])
    print(counter)
```

### Interacting with a contract

For every public and external function in Solidity source code, Woke generates a Python method in `pytypes`.
These methods can be used to interact with a deployed contract. Generated methods accept the same arguments as the corresponding Solidity functions.
Additional keyword arguments can configure the execution of a function like with the `deploy` method.

```python
from woke.testing import *
from pytypes.contracts.Counter import Counter

@default_chain.connect()
def test_counter():
    owner = default_chain.accounts[0]
    other = default_chain.accounts[1]

    counter = Counter.deploy(from_=owner)

    counter.increment(from_=other)
    assert counter.count() == 1

    # setCount can only be called by the owner
    counter.setCount(10, from_=owner)
    assert counter.count() == 10

    # this will fail because the sender account is not the owner
    with must_revert():
        counter.setCount(20, from_=other)
    assert counter.count() == 10
```
