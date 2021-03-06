# ckb-zkp

Smart contracts that run as zero-knowledge proof verifiers on the [Nervos CKB chain](https://www.nervos.org/). CKB developers and users can implement various complex zero-knowledge verification processes through the simplest contract invocation. Cooperate with zkp-toolkit to complete off-chain prove and on-chain verify.

This project is part of *zkp-toolkit-ckb* and is supported by the Nervos Foundation. Check out the [original proposal](https://talk.nervos.org/t/secbit-labs-zkp-toolkit-ckb-a-zero-knowledge-proof-toolkit-for-ckb/4254) and [grant announcement](https://medium.com/nervosnetwork/three-new-ecosystem-grants-awarded-892b97e8bc06).

## Table of contents

- [ckb-zkp](#ckb-zkp)
  - [Table of contents](#table-of-contents)
  - [Prerequisites](#prerequisites)
  - [Build contracts](#build-contracts)
    - [Enable `debug!` macro in release mode](#enable-debug-macro-in-release-mode)
  - [Tests](#tests)
    - [Prerequises for testing](#prerequises-for-testing)
    - [Run tests](#run-tests)
    - [Invoking the contract on-chain](#invoking-the-contract-on-chain)
    - [Debugging the `capsule` itself (Temporary feature)](#debugging-the-capsule-itself-temporary-feature)
  - [Optimizations & Benchmarks](#optimizations--benchmarks)
    - [Binary optimization](#binary-optimization)
    - [Curve benchmark](#curve-benchmark)
    - [Impacts of disabling `zkp-toolkit` feature](#impacts-of-disabling-zkp-toolkit-feature)
    - [Further optimizations](#further-optimizations)
  - [Troubleshooting](#troubleshooting)
    - [`capsule` complained `error: Can't found capsule.toml, current directory is not a project`](#capsule-complained-error-cant-found-capsuletoml-current-directory-is-not-a-project)
    - [I can't see any output in the ckb's log on dev vhain.](#i-cant-see-any-output-in-the-ckbs-log-on-dev-vhain)
    - [The test can't find contract binary/proof file/vk file.](#the-test-cant-find-contract-binaryproof-filevk-file)
    - [How is the project mounted into the Docker container?](#how-is-the-project-mounted-into-the-docker-container)
  - [Acknowledgement](#acknowledgement)
  - [Security](#security)
  - [References](#references)
  - [License](#license)

## Prerequisites

1. Install the development framework [`capsule`](https://github.com/jjyr/capsule) by jjy, a developer of the Nervos network. Access its [wiki page](https://github.com/nervosnetwork/capsule/wiki) for more details about `capsule`.

<!-- TODO update capsule revision -->

```sh
cargo install capsule --git https://github.com/jjyr/capsule.git --rev=089a5505
```

`capsule` is under development and not stable, so please specify the revision when installing.

2. Pull the [docker image from jjy](https://hub.docker.com/r/jjy0/ckb-capsule-recipe-rust), which is used to build contracts.

   ```sh
   docker pull jjy0/ckb-capsule-recipe-rust:2020-6-2
   ```

3. Deploy a ckb dev chain. See https://docs.nervos.org/dev-guide/devchain.html for guidance.

4. Add the local dependency (optional) and specify the revision of dependency (necessary).

   ATTENTION: **"Adding the local dependency" is not necessary when the `zkp-toolkit` repo is available on Github. Only use local dependency on development, especially when developing `zkp-toolkit`.**

   - `zkp-toolkit` is available on Github or crates.io, and you don't need to modify `zkp-toolkit`.

     Simply use the git URL or version tag in the manifest of the contract. If you want to modify the `zkp-toolkit` library, see the section below.

     Specify the revision of dependency `zkp-toolkit`.

     ```toml
     # File: ./contracts/ckb-zkp/Cargo.toml
     [dependencies]
     zkp = { git = "https://github.com/sec-bit/zkp-toolkit.git", rev = "3e5ff9db7a", default-features = false, features = [
         "groth16",
         "bn_256",
         "bls12_381",
     ] }
     ```

   - `zkp-toolkit` is not available on Github or crates.io, or you want to modify `zkp-toolkit`.

     Pull the dependency `zkp-toolkit` into _dependencies_ folder like _./dependencies/zkp-toolkit_, and checkout the revision:

     ```sh
     # At ./dependencies/zkp-toolkit
     git checkout 3e5ff9db7a
     ```

     Specify local dependency in contract's Cargo manifest:

     ```toml
     # File: ./contracts/ckb-zkp/Cargo.toml
     [dependencies]
     zkp = { path = "../../dependencies/zkp-toolkit", default-features = false, features = [
         "groth16",
         "bn_256",
         "bls12_381",
     ] }
     ```

     Reason: During early development, the dependency `zkp-toolkit` is not available via a public git URL, so we can only access this dependency via a local path.

## Build contracts

Like Cargo, you can choose to build the contract in **dev** mode or **release** mode. The product under release mode is suitable for deployment with a reasonable size and execution consumption, and, `debug!` macro is disabled. Dev mode product allows you to use `debug!` macro to print logs in ckb log, but on the cost of larger binary size and execution cycles. The product resides in _./build/[release|debug]/ckb-zkp_.

Note that: all the `capsule` commands should be executed at the project root

```sh
# At project root
# Dev mode, enable debug! macro but result in bloated size.
capsule build
# Release mode. Slim, no debug!.
capsule build --release
```

### Enable `debug!` macro in release mode

**In `ckb-std` version 0.2.2 and newer, `debug!` macro is disabled in release mode**. If you still want to enable `debug!` macro in **release** mode, insert `debug-assertions = true` under `[profile.release]` in `contracts/ckb-zkp/Cargo.toml`.

## Tests

### Prerequisites for testing

1. Go to _./dependencies/zkp-toolkit/cli_ and generate a vk file and a proof file using `zkp-toolkit`. Then put these files into anywhere of the project folder, and assign the positions and names of the files in _./tests/src/tests.rs_. The default location and names suit the local dependency pattern.

   Use groth16 scheme & bn_256 curve:

   1. Complete trusted-setup:

      ```sh
      # ./dependencies/zkp-toolkit/cli
      cargo run --bin trusted-setup mimc
      ```

   2. Prove the secret string.

      ```sh
      # ./dependencies/zkp-toolkit/cli
      cargo run --bin zkp-prove mimc --string=iamsecret
      ```

      When successful, it will create a proof file at proofs_files.

   3. (Optional) Do the verification.

      ```sh
      # ./dependencies/zkp-toolkit/cli
      cargo run --bin zkp-verify mimc proofs_files/mimc.groth16-bn_256.proof
      ```

   Use groth16 as scheme and bls12_381 as curve:

   ```sh
   # ./dependencies/zkp-toolkit/cli
   # trusted-setup
   cargo run --bin trusted-setup mimc groth16 bls12_381
   # Prove the secret string
   cargo run --bin zkp-prove mimc groth16 bls12_381 --string=iamsecret
   # Verification.
   cargo run --bin zkp-verify mimc groth16 bls12_381 proofs_files/mimc.groth16-bls12_381.proof
   ```

   See https://aciclo.net/zkp/zkp-toolkit#cli-command for further help.

### Run tests

**Make sure vk file(s) and proof file(s) are prepared and can be found by the test suit.**

Then type the following command.

ATTENTION: If you build the contract with `--release` flag, you should run tests with `CAPSULE_TEST_ENV=release`.

the flag `--test-threads 1` after `--` is used to ensure `debug!` outputs print in order.

```sh
# At project root
# Dev mode contracts.
cargo test -p tests --tests -- --nocapture --test-threads 1
# Release mode contracts.
CAPSULE_TEST_ENV=release cargo test -p tests --tests -- --nocapture
```

In file _./tests/src/tests.rs_, you can uncomment the `#[ignore]` attribute before a test function to omit it. Or specify the test function name to filter others.

```sh


## Deployment

`Capsule` brings out-of-box contract deploying and migrating. It works for development and test on dev chain. To deploy a contract you have just cooked, you need:

- A running ckb client on the local machine or the net.
- A ckb-cli executable. `capsule` uses ckb-cli to interact with ckb client.
- An account with sufficient CKBs for deployment (1 Byte of contract binary will consume 1 CKB. The transaction body will also take some extra CKBs, but not much). This account should be imported into ckb-cli.
- A deployment manifest _./deployment.toml_, which assigns the contract binary and cell lock-arg.

When everything needed is met, you should theoretically be able to deploy the contract. Use the command below to launch the transaction, and note that commonly the `<ADDRESS>` is a 46-bit alphanumeric string (Starting with `ckt1` if you use a test net or dev chain).

```shell
# At project root
capsule deploy --address <ADDRESS>
```

### Invoking the contract on-chain

TODO: No ready-to-use gear for invoking, use `ckb-cli`, or an SDK to build a transaction to invoke the contract.

### Debugging the `capsule` itself (Temporary feature)

You can use the **master** branch of `capsule` and the following commands to track the panics.

```shell
# At project root
RUST_LOG=capsule=trace capsule deploy --address <ADDRESS>
```

## Optimizations & Benchmarks

### Binary optimization

In ckb, the costs come from the size of the built transaction. Heavier in size means higher in cost, while running cost (total instructions executed) has little impact on the deployment and invocation of contracts. So several compiling options are used to try to reduce the contract binary size.

- To build in release mode, this is enabled by default.
- LTO
- Strip
- `opt-level`
- `codegen-units`

To use LTO, `opt-level` and `codegen-units`, modify _Cargo.toml_:

```toml
# File: contracts/ckb-zkp/Cargo.toml
[profile.release]
overflow-checks = true
# lto: true, "thin", false(default)
lto = true
# opt-level: 0(default), 1, 2, 3, "s", "z"
opt-level = "z"
# codegen-units: greater than 0, default 16
codegen-units = 1
```

To strip the binary, use `rustflags = "-C link-arg=-s"` in cargo config, which is a default option in Capsule with release compiling mode.

We will not try to explain what each option means (Explained in _The Cargo Book_), but list the size and running cost of the contract binaries under different option combinations.

Test setup:

- Release mode;
- stripped;
- using `jjy0/ckb-capsule-recipe-rust:2020-6-2` to build and test and measure running costs;
- using scheme groth16 and curve bn_256;
- `ckb-std` 0.3.0;
- `zkp-toolkit` revision 3bfcda742a;
- `ckb-tool` and `ckb-testtool` version 0.0.1.

| LTO     | `opt-level` | `codegen-units` | Binary size(Byte) | Execution cost (cycles) |
| ------- | ----------- | --------------- | ----------------- | ----------------------- |
| not set | not set     | not set         | 496472            | 94548494                |
| true    | not set     | not set         | 418576            | 99437440                |
| not set | “z”         | not set         | 217944            | 1145224343              |
| true    | “z”         | not set         | 176912            | 212607081               |
| not set | “z”         | 1               | 136024            | 1180836966              |
| true    | “z”         | 1               | 115472            | 222454238               |
| true    | “s”         | 1               | 213776            | 158433197               |

Here comes a rough result:

- Enabling LTO, use `opt-level = "z"`, `codegen-units = 1` for minimun binary size, with a large cycle consumption.
- Enabling LTO, use `opt-level = "s"`, `codegen-units = 1` will increase the binary size, and decrease the cycle consumption.

### Curve benchmark

Currently, we use two different curves in proving and verifying, so we performed a simple benchmark on execution costs separately.

Test setup:

- Release mode;
- stripped;
- `LTO = true`, `codegen-units = 1`;
- using `jjy0/ckb-capsule-recipe-rust:2020-6-2` to build and test and measure running costs;
- using scheme groth16 and curve bn_256;
- `ckb-std` 0.3.0;
- `zkp-toolkit` revision 3bfcda742a;
- `ckb-tool` and `ckb-testtool` version 0.0.1.

| Curve     | `opt-level` | Binary size(Byte) | Execution cost (cycles) |
| --------- | ----------- | ----------------- | ----------------------- |
| bn_256    | "z"         | 115472            | 222454238               |
| bn_256    | "s"         | 213776            | 158433197               |
| bls12_381 | "z"         | 115472            | 354845813               |
| bls12_381 | "s"         | 213776            | 314480688               |

### Impacts of disabling `zkp-toolkit` feature

Different curves are enabled as crate features in the contract. The number of features enabled will impact the binary size and execution cost.

Test setup:

- Release mode;
- stripped;
- `LTO = true`, `codegen-units = 1`;
- using `jjy0/ckb-capsule-recipe-rust:2020-6-2` to build and test and measure running costs;
- using scheme groth16;
- `ckb-std` 0.3.0;
- `zkp-toolkit` revision 3bfcda742a;
- `ckb-tool` and `ckb-testtool` version 0.0.1.

| Feature enabled   | Binary size(Byte) | Curve using | Execution cost (cycles) | Execution cost Diff |
| ----------------- | ----------------- | ----------- | ----------------------- | ------------------- |
| None              | 29456             | N/A         | N/A                     | N/A                 |
| bn_256            | 74512             | bn_256      | 222407799               | -46439              |
| bls12_381         | 74512             | bls12_381   | 354757562               | -88251              |
| bls12_381, bn_256 | 115472            | bn_256      | 222454238               | 0                   |
| bls12_381, bn_256 | 115472            | bls12_381   | 354845813               | 0                   |

### Further optimizations

We have accomplished the main goal we set for the Milestone-I of the [zkp-toolkit-ckb](https://talk.nervos.org/t/secbit-labs-zkp-toolkit-ckb-a-zero-knowledge-proof-toolkit-for-ckb/4254), which was a simple on-chain verifier for CKB. The proof-of-concept smart contract code shows that we can make a usable zkp verifier for CKB with pure Rust without modifying the underlying chain. This also gives us a baseline on the performance of zkp verifiers for CKB-VM.

We'll implement more zkp verifiers in the following milestones, looking at reducing the binary size and [execution](https://xuejie.space/2020_03_03_introduction_to_ckb_script_programming_performant_wasm/) [cost](https://xuejie.space/2020_04_09_language_choices/), as well as the best practice to integrate with other contracts.

## Troubleshooting

### `capsule` complained `error: Can't found capsule.toml, current directory is not a project`

All the commands executed by `capsule` should be executed under the project root.

### I can't see any output in the CKB's log on dev chain.

Modify ckb's configuration as below:

```toml
# File: ckb.toml of your chain.
[logger]
filter = "info,ckb-script=debug"
```

### The test can't find contract binary/proof file/vk file.

Make sure you **build and test the contract in the same mode** (dev or release, specified by flag `--release`).

```sh
# At project root
capsule build && capsule test
# Or
capsule build --release && capsule test --release
```

As capsule executes building and testing in docker, the absolute path may not work as expected, so **use relative path**. And currently, the Capsule (jjyr/capsule revision 2f9513f8) mount the whole project folder into docker, so any relative location inside the project folder is allowed.

### How is the project mounted into the Docker container?

In the fork of `jjyr/capsule` revision `2f9513f8`, `capsule` mounts the project folder into the container with path _/code_. But in the main source `nervosnetwork/capsule`, `capsule` may only mount the contract folder into the container. As docker is used, the absolute path is not recommended.

## Acknowledgement

- Many thanks to [jjy](https://github.com/jjyr), a developer of [nervosnetwork](https://github.com/nervosnetwork), for his selfless help and advice on this project.

## Security

This project is still under active development and is currently being used for research and experimental purposes only, please **DO NOT USE IT IN PRODUCTION** for now.

## References

<!-- TODO -->

## License

This project is licensed under either of

- Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or
  http://www.apache.org/licenses/LICENSE-2.0)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or
  http://opensource.org/licenses/MIT)

at your option.
