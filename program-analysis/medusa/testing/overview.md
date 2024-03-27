# Testing Overview

## What Are Invariants?

Invariants are properties or conditions that should always hold true during the execution of a program. They are used to detect errors or unexpected behavior by asserting that these properties remain valid throughout the test.
Examples of invariants in a token contract would be:

- The balance of an individual account should never be negative.
- The total balance of all accounts in a system should never be more than the total supply of tokens.

## Testing Modes

Medusa supports three main testing modes:

- **Property testing:** Checks if certain properties of the contract hold true under all possible inputs. Property tests are functions prefixed with a specified prefix (e.g., `fuzz_`) that take no arguments and return a boolean indicating success.
- **Assertion testing:** Verifies that specific assertions within the contract hold true during execution. Assertion tests are functions that contain `assert` statements or cause the EVM to panic.
- **Optimization testing:** Attempts to maximize the return value of a given function. Optimization tests are functions prefixed with a specified prefix (e.g., `optimize_`) that take no arguments and return an integer.

Check out [this](./testing_modes.md) chapter for more on medusa's testing modes.

## Testing Approaches

The approach you take to testing depends on your requirements and the codebase to be tested. Some common testing approaches include:

- **Internal Testing:** Properties are defined within the contract to test, providing complete access to the internal state of the system. This approach is useful for simpler contracts with a single entry point and simpler initialization.
- **External Testing:** Properties are tested using external calls from a different contract, allowing access to external/public variables or functions. Several alternatives, such as contract wrappers and TestAllContracts, are discussed for handling transactions.
- **Partial Testing:** Testing some of the components of the complete deployed system, ignoring or abstracting uninteresting parts such as standard ERC20 tokens or oracles, is discussed. Isolated testing, function override, and model testing are mentioned as approaches for partial testing.

Check out the [testing approaches chapter](./testing_approaches.md) chapter for more on the aforementioned approaches to testing.

## Running Tests

Medusa can be run with one or more testing modes enabled via command-line flags or a configuration file. A fuzzing campaign can be started using the `medusa fuzz` command. For more detail on this, check out the [../cli/fuzz.md](CLI chapter).
