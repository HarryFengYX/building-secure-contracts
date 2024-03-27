## Writing and Testing System-Level Invariants with Medusa

This chapter will walk you through writing system-level fuzz tests for an ERC20 token as discussed in the [invariants chapter](./invariants.md#system-level-invariants).

### Introduction

System-level invariants are properties that must remain true throughout the execution of a system. This means that no point in time should these properties be broken. Before writing fuzz tests, these properties should first be identified and better still, written in plain text. Only after that do we start writing the fuzz tests.

### Writing System-Level Invariants as Property Tests

Property tests are represented as functions within a Solidity contract whose names are prefixed with a prefix specified by the `testPrefixes` configuration option (`fuzz_` is the default test prefix). Additionally, they must take no arguments and return a `bool` indicating if the test succeeded.

For example, the following property test checks that no user's token balance exceeds the total number of tokens in the contract:

```solidity
contract TestContract is Token {
  function fuzz_userBalanaceShouldNotExceedTotalSupply() public returns (bool) {
    return balances[msg.sender] <= totalSupply;
  }
}
```

If, during the fuzzing campaign, the balance of any of the sender addresses exceeds the total supply of tokens, the property test will fail, and medusa will report the call sequence that broke the property.

### Writing System-Level Invariants as Assertion Tests

Assertion tests check to see if a given call sequence can cause an assertion failure. An assertion failure is encountered when an `assert(x)` statement returns `false`.

For example, the following assertion test checks that the token's decimals do not change during fuzzing:

```solidity
contract TestContract {
  Token token;

  constructor() {
    token = new Token("MyToken", "MT", 18);
  }

  function testTokenDecimalsDoesNotChange() public {
    assert(token.decimals() == 18);
  }
}
```

The assert statement and the test will fail if `token.decimals()` has a value other than `18` at any time during the fuzzing campaign. The call sequence that led to the assertion failure will be reported by Medusa.
