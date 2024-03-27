## [Testing Approaches](#testing-approaches)

Unlike standard binaries, testing smart contracts can be quite complex due to multiple accounts interacting with one or more entry points. Medusa takes care not to violate some fundamental underlying assumptions of transactions that are impossible in Ethereum (e.g. account with unlimited amount of ETH, sending a transaction from the zero address etc.), even though a fuzzer can mimic the Ethereum Virtual Machine and potentially use any account with any feature. For this reason, understanding the system to be tested and the manner in which transactions will be emulated is essential. The testing strategy falls into a number of categories. We will start with two of them, internal and external.

**Table of contents:**

- [Internal Testing](#internal-testing)
- [External Testing](#external-testing)
- [Partial Testing](#partial-testing)

### [Internal Testing](#internal-testing)

In this testing approach, properties are defined within the contract to test, giving complete access to the internal state of the system.

```
contract InternalTest is System {
    function fuzz() public returns (bool) {
        return stateVar > X;
    }
}
```

With this approach, Medusa creates transactions between the target contract and a simulated account. This approach works especially well for smaller contracts with a single entry point and no complicated startup requirements. Because properties have access to the internal state of the system, they may also be simpler to write.

### [External Testing](#external-testing)

In the external testing approach, properties are tested using external calls from a different contract. Properties are only allowed to access external/public variables or functions.

```
contract ExternalTest {
    constructor() public {
        addr = address(0x1234);
    }

    function fuzz_state_greater_than_X() public returns (bool) {
        return System(addr).stateVar() > X;
    }
}
```

When handling contracts that call for external initialization, this testing strategy comes in handy. It's crucial to pay attention to how Medusa handles the transactions, though. Running Medusa directly on this will prevent any code execution from the contract from being tested, as `ExternalTest` provides no extra methods. Consequently, there won't be any calls to the functions in the contract that need to be tested — just the actual properties. In this case, there are several alternatives:

**Contract wrapper**: Define specific operations to "wrap" the system for testing. For each operation that we want Medusa to execute in the system to test, we add one or more functions that perform an external call to it.

```
contract ExternalTest {
    constructor() public {
        // addr = ...;
    }

    function method(...) public returns (...) {
        return System(addr).method();
    }

    function fuzz_state_greater_than_X() public returns (bool) {
        return System(addr).stateVar() > X;
    }
}
```

There are two important points to consider with this approach:

- The sender of each transaction will be the `ExternalTest` contract, instead of the simulated Medusa sender addresses (e.g., `0x10000`, ..). This means that the real address interacting with the system will be the `External` contract's address, rather than one of the Medusa sender addresses. Please take special care if this contract needs to be provided ETH or tokens.
- This approach is manual and can be time-consuming if there are many function operations. However, it can be useful when Medusa needs help calculating a value that cannot be randomly sampled:

```
contract ExternalTest {
    // ...

    function methodUsingF(..., uint256 x) public returns (...) {
        return System(addr).method(.., f(x));
    }

    ...
}
```

**TestAllContracts**: Medusa can perform direct calls to every contract (including dynamically deployed ones) if the `TestAllContracts` config option is enabled. This means that using it does not require wrapped calls. However, since every deployed contract can be called, unintended effects may occur. For example, if we have a property to ensure that the amount of tokens is limited:

```
contract ExternalTest {
    constructor() {
        addr = ...;
        MockERC20(...).mint(...);
    }

    function fuzz_limited_supply() public returns (bool) {
        return System(addr).balanceOf(...) <= X;
    }

    ...
}
```

Using "mock" contracts for tokens (e.g., MockERC20) could be an issue because Medusa could call functions that are public but are only supposed to be used during the initialization, such as `mint`.

Another benefit of using this approach is that it forces the developer or auditor to write properties using public data. If an essential property cannot be defined using public data, it could indicate that users or other contracts will not be able to easily interact with the system to perform an operation or verify that the system is in a valid state.

### [Partial Testing](#partial-testing)

Ideally, testing a smart contract system uses the complete deployed system, with the same parameters that the developers intend to use. Testing with the real code is always preferred, even if it is slower than other methods (except for cases where it is extremely slow). However, there are many cases where, despite the complete system being deployed, it cannot be simulated because it depends on off-chain components (e.g., a token bridge). In these cases, alternative solutions must be implemented.

With partial testing, we test some of the components, ignoring or abstracting uninteresting parts such as standard ERC20 tokens or oracles. There are several ways to do this.

**Isolated testing**: If a component is adequately abstracted from the rest of the system, testing it can be easy. This method is particularly useful for testing stateless properties found in components that compute mathematical operations, such as mathematical libraries.

**Function override**: Solidity allows for function overriding, used to change the functionality of a code segment without affecting the rest of the codebase. We can use this to disable certain functions in our tests to allow testing with Medusa:

```
contract InternalTestOverridingSignatures is System {
    function verifySignature(...) public override returns (bool) {
        return true; // signatures are always valid
    }

    function fuzz_state_greater_than_X() public returns (bool) {
        executeSomethingWithSignature(...);
        return stateVar > X;
    }
}
```

**Model testing**: If the system is not modular enough, a different approach is required. Instead of using the code as is, we will create a "model" of the system in Solidity, using mostly the original code. Although there is no defined list of steps to build a model, we can provide a generic example. Suppose we have a complex system that includes this piece of code:

```
contract System {
    ...

    function calculateSomething() public returns (uint256) {
        if (booleanState) {
            stateSomething = (uint256State1 * uint256State2) / 2 ** 128;
            return stateSomething / uint128State;
        }

        ...
    }
}
```

Where `boolState`, `uint256State1`, `uint256State2`, and `stateSomething` are state variables of our system to test. We will create a model (e.g., copy, paste, and modify the original code in a new contract), where each state variable is transformed into a parameter:

```
contract SystemModel {
    function calculateSomething(bool boolState, uint256 uint256State1, ...) public returns (uint256) {
        if (boolState) {
            stateSomething = (uint256State1 * uint256State2) / 2 ** 128;
            return stateSomething / uint128State;
        }

        ...
    }
}
```

At this point, we should be able to compile our model without any dependency on the original codebase (everything necessary should be included in the model). We can then insert assertions to detect when the returned value exceeds a certain threshold.

While developers or auditors may be tempted to quickly create tests using this technique, there are certain disadvantages when creating models:

- The tested code can be very different from what we want to test. This can either introduce unreal issues (false positives) or hide real issues from the original code (false negatives). In the example, it is unclear if the state variables can take arbitrary values.
- The model will have limited value if the code changes since any modification to the original model will require manually rebuilding the model.
