# Writing Rust Contracts on CasperLabs

## Smart Contracts
This section explains step by step how to write a new smart contract.

### Basic Smart Contract

The CasperLabs VM executes a smart contract by calling the `call` function specified in the contract. If the function is missing, the smart contract is not valid. The simplest possible example is an empty `call` function.
```rust
#[no_mangle]
pub extern "C" fn call() {}
```
The `#[no_mangle]` attribute prevents the compiler from changing (mangling) the function name when converting to the binary format of Wasm. Without it, the VM exits with the error message: `Module doesn't have export call`.

### Using Error Codes
The CasperLabs VM supports error codes in smart contracts. A contract can stop execution and exit with a given error via the [`runtime::revert`](https://docs.rs/casperlabs-contract/latest/casperlabs_contract/contract_api/runtime/fn.revert.html) function:
```rust
use casperlabs_contract::contract_api::runtime;
use casperlabs_types::ApiError;

#[no_mangle]
pub extern "C" fn call() {
    runtime::revert(ApiError::PermissionDenied)
}
```

CasperLabs has [several built-in error variants](https://docs.rs/casperlabs-types/latest/casperlabs_types/enum.ApiError.html#mappings), but it's possible to create a custom set of error codes for your smart contract. These can be passed to [`runtime::revert`](https://docs.rs/casperlabs-contract/latest/casperlabs_contract/contract_api/runtime/fn.revert.html) via [`ApiError::User(<your error code>)`](https://docs.rs/casperlabs-types/latest/casperlabs_types/enum.ApiError.html#variant.User).

When a contract exits with an error code, the exit code is visible in the Block Explorer.

### Arguments
It's possible to pass arguments to smart contracts. To leverage this feature, use [`runtime::get_arg`](https://docs.rs/casperlabs-contract/latest/casperlabs_contract/contract_api/runtime/fn.get_arg.html). The helper function [`unwrap_or_revert_with`](https://docs.rs/casperlabs-contract/latest/casperlabs_contract/unwrap_or_revert/trait.UnwrapOrRevert.html#tymethod.unwrap_or_revert_with) is available on `Option` and `Result` when importing [`unwrap_or_revert::UnwrapOrRevert`](https://docs.rs/casperlabs-contract/latest/casperlabs_contract/unwrap_or_revert/trait.UnwrapOrRevert.html).
```rust
use casperlabs_contract::{
    contract_api::runtime,
    unwrap_or_revert::UnwrapOrRevert,
};

#[no_mangle]
pub extern "C" fn call() {
    let value: String = runtime::get_arg(0)
        // Unwrap the `Option`, returning an error if there was no argument supplied.
        .unwrap_or_revert_with(ApiError::MissingArgument)
        // Unwrap the `Result` containing the deserialized argument or return an error if there was
        // a deserialization error.
        .unwrap_or_revert_with(ApiError::InvalidArgument);
}
```

### Storage
Saving and reading values to and from the blockchain is a manual process in CasperLabs. It requires more code to be written, but also provides a lot of flexibility. The storage system works similarly to a file system in an operating system.  Let's say we have a string `"Hello CasperLabs"` that needs to be saved. To do this, use the text editor, create a new file, paste the string in and save it under a name in some directory. The pattern is similar on the CasperLabs blockchain. First you have to save your value to the memory using [`storage::new_turef`](https://docs.rs/casperlabs-contract/latest/casperlabs_contract/contract_api/storage/fn.new_turef.html). This returns a reference to the memory object that holds the `"Hello CasperLabs"` value. You could use this reference to update the value to something else. It's like a file. Secondly you have to save the reference under a human-readable string using [`runtime::put_key`](https://docs.rs/casperlabs-contract/latest/casperlabs_contract/contract_api/runtime/fn.put_key.html). It's like giving a name to the file. The following function implements this scenario:
```rust
const KEY: &str = "special_value";

fn store(value: String) {
    // Store `value` under a new unforgeable reference.
    let value_ref = storage::new_turef(value);

    // Wrap the unforgeable reference in a `Key`.
    let value_key: Key = value_ref.into();

    // Store this key under the name "special_value" in context-local storage.
    runtime::put_key(KEY, value_key);
}
```
After this function is executed, the context (Account or Smart Contract) will have the content of the `value` stored under `KEY` in its named keys space. The named keys space is a key-value storage that every context has. It's like a home directory.

### Final Smart Contract
The code below is the simple contract generated by [`cargo-casperlabs`](https://github.com/CasperLabs/CasperLabs/tree/master/execution-engine/cargo-casperlabs) (found in `contract/src/lib.rs` of a project created by the tool). It reads an argument and stores it in the memory under a key named `"special_value"`.
```rust
#![cfg_attr(
    not(target_arch = "wasm32"),
    crate_type = "target arch should be wasm32"
)]

use casperlabs_contract::{
    contract_api::{runtime, storage},
    unwrap_or_revert::UnwrapOrRevert,
};
use casperlabs_types::{ApiError, Key};

const KEY: &str = "special_value";

fn store(value: String) {
    // Store `value` under a new unforgeable reference.
    let value_ref = storage::new_turef(value);

    // Wrap the unforgeable reference in a value of type `Key`.
    let value_key: Key = value_ref.into();

    // Store this key under the name "special_value" in context-local storage.
    runtime::put_key(KEY, value_key);
}

// All session code must have a `call` entrypoint.
#[no_mangle]
pub extern "C" fn call() {
    // Get the optional first argument supplied to the argument.
    let value: String = runtime::get_arg(0)
        // Unwrap the `Option`, returning an error if there was no argument supplied.
        .unwrap_or_revert_with(ApiError::MissingArgument)
        // Unwrap the `Result` containing the deserialized argument or return an error if there was
        // a deserialization error.
        .unwrap_or_revert_with(ApiError::InvalidArgument);

    store(value);
}
```

## Tests
As part of the CasperLabs local environment we provide the in-memory virtual machine you can run your contract against. The testing framework is designed to be used in the following way:
1. Initialize the context.
2. Deploy or call the smart contract.
3. Query the context for changes and assert the result data matches expected values.

### TestContext
A [`TestContext`](https://docs.rs/casperlabs-engine-test-support/latest/casperlabs_engine_test_support/struct.TestContext.html) provides a virtual machine instance. It should be a mutable object as we will change its internal data while making deploys. It's also important to set an initial balance for the account to use for deploys.
```rust
const MY_ACCOUNT: [u8; 32] = [7u8; 32];

let mut context = TestContextBuilder::new()
    .with_account(MY_ACCOUNT, U512::from(128_000_000))
    .build();
```
Account is type of `[u8; 32]`. Balance is type of `U512`.

### Run Smart Contract
Before we can deploy the contract to the context, we need to prepare the request. We call the request a [`Session`](https://docs.rs/casperlabs-engine-test-support/latest/casperlabs_engine_test_support/struct.Session.html). Each session call should have 4 elements:
- Wasm file path.
- List of arguments.
- Account context of execution.
- List of keys that authorize the call. See: TODO insert keys management link.
```rust
let VALUE: &str = "hello world";
let session_code = Code::from("contract.wasm");
let session_args = (VALUE,);
let session = SessionBuilder::new(session_code, session_args)
    .with_address(MY_ACCOUNT)
    .with_authorization_keys(&[MY_ACCOUNT])
    .build();
context.run(session);
```
Executing `run` will panic if the code execution fails.

### Query and Assert
The smart contract we deployed creates a new value `"hello world"` under the key `"special_value"`. Using the `query` function it's possible to extract this value from the blockchain.
```rust
let KEY: &str = "special_value";
let result_of_query: Result<Value, Error> = context.query(MY_ACCOUNT, &[KEY]);
let returned_value = result_of_query.expect("should be a value");
let expected_value = Value::from_t(VALUE.to_string()).expect("should construct Value");
assert_eq!(expected_value, returned_value);
```
Note that the `expected_value` is a `String` type lifted to the `Value` type. It was also possible to map `returned_value` to the `String` type.

### Final Test
The code below is the simple test generated by [`cargo-casperlabs`](https://github.com/CasperLabs/CasperLabs/tree/master/execution-engine/cargo-casperlabs) (found in `tests/src/integration_tests.rs` of a project created by the tool).
```rust
#[cfg(test)]
mod tests {
    use casperlabs_engine_test_support::{Code, Error, SessionBuilder, TestContextBuilder, Value};
    use casperlabs_types::U512;

    const MY_ACCOUNT: [u8; 32] = [7u8; 32];
    // define KEY constant to match that in the contract
    const KEY: &str = "special_value";
    const VALUE: &str = "hello world";

    #[test]
    fn should_store_hello_world() {
        let mut context = TestContextBuilder::new()
            .with_account(MY_ACCOUNT, U512::from(128_000_000))
            .build();

        // The test framework checks for compiled Wasm files in '<current working dir>/wasm'.  Paths
        // relative to the current working dir (e.g. 'wasm/contract.wasm') can also be used, as can
        // absolute paths.
        let session_code = Code::from("contract.wasm");
        let session_args = (VALUE,);
        let session = SessionBuilder::new(session_code, session_args)
            .with_address(MY_ACCOUNT)
            .with_authorization_keys(&[MY_ACCOUNT])
            .build();

        let result_of_query: Result<Value, Error> = context.run(session).query(MY_ACCOUNT, &[KEY]);

        let returned_value = result_of_query.expect("should be a value");

        let expected_value = Value::from_t(VALUE.to_string()).expect("should construct Value");
        assert_eq!(expected_value, returned_value);
    }
}

fn main() {
    panic!("Execute \"cargo test\" to test the contract, not \"cargo run\".");
}
```

## WASM File Size
We encourage shrinking the WASM file size as much as possible. Smaller deploys cost less and save the network bandwidth. We recommend reading [Shrinking .wasm Code Size](https://rustwasm.github.io/docs/book/reference/code-size.html) chapter of [The Rust Wasm Book](https://rustwasm.github.io/docs/book/).