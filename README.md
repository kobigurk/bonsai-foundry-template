# Bonsai Foundry Template

Starter template for writing an application using [Bonsai].

This repository implements an application on Ethereum utilizing Bonsai as a coprocessor to the smart contract application.
It provides a starting point for building powerful new applications on Ethereum that offload computationally intensive
(or difficult to implement) tasks to be proven by the [RISC Zero] [zkVM], with verifiable results sent to your Ethereum contract.

*For a 60 second overview of how this template and off-chain computation with Bonsai work, [check out the video here](https://www.youtube.com/watch?v=WDS8X8H9mIk).*

## Dependencies

1. [Rust and Cargo](https://rustup.rs)

    ```bash
    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
    ```

2. [Foundry](https://getfoundry.sh/)

    ```bash
    curl -L https://foundry.paradigm.xyz | bash
    ```

***Tip:*** *If you're installing Foundry for the first time, be sure to quit your terminal before reopening it and running the `foundryup` command.*

## Project Setup

1. Use Foundry to create a new project from this template at a location of your choosing (this example command will create it at `./my-project`)

    ```bash
    forge init -t risc0/bonsai-foundry-template ./my-project
    ```

## Get Started

Get started writing your application by modifying these key files:

* Replace `contracts/BonsaiStarter.sol` with your on-chain application logic.
* Replace `methods/guest/src/bin/fibonacci.rs` with your [zkVM guest program].

Associated build configuration files and tests are discussed along with the [project structure](#project-structure) below.

## Test Your Project

With this Foundry template, you can write a zkVM Rust program in the `/methods/guest` directory which Solidity contracts in  `/contracts` can call into and in return receive a proof or 'receipt' of execution.

### Test your zkVM program

To check if your zkVM program will compile, and generate any errors if not, run the Rust compiler with

```bash
cargo build
```

If you've written tests in your zkVM progam, run them with

```bash
cargo test
```

***Tip:*** *To learn more about our RISC-V zkVM [visit the docs](https://dev.risczero.com/zkvm) or for a thorough walkthrough, follow the [Factors Tutorial here](https://github.com/risc0/risc0/tree/main/examples/factors#tutorial).*

### Test your solidity integration with the zkVM

To test both your Solidity contracts and their interaction with your zkVM program, run

```bash
forge test
```

***Tip:*** *To learn more about Foundry's `forge` command and the other helpful utilities Foundry provides, visit their docs: https://book.getfoundry.sh/forge/tests.*

### Deploy your project on a local network

You can deploy your contracts and run an end-to-end test or demo as follows:

1. Start an anvil instance, if you want a local testnet, by running:

    ```bash
    anvil
    ```

    Once anvil is started, keep it running in the terminal, and switch to a new terminal.

2. Deploy the `BonsaiRelay` contract by running:

    ```bash
    RISC0_DEV_MODE= forge script script/Deploy.s.sol --rpc-url http://localhost:8545 --broadcast
    ```

    Check the logs for the address of the `BonsaiRelay` contract.

3. Start the Bonsai Ethereum Relay by running:

    ```bash
    RISC0_DEV_MODE= cargo run --bin bonsai-ethereum-relay-cli -- run --relay-address 0x5FbDB2315678afecb367f032d93F642f64180aa3
    ```

    The relay will keep monitoring the chain for callback requests and relay their result back after computing them.
    You should keep this terminal instance running the relay in the foreground and switch to a new terminal.
    When setting the env variable `RISC0_DEV_MODE`, the relay will start a mock version of Bonsai that will only return a journal of the execution without any cryptographic proof. 
    If needed, you should modify the environment variables to reflect your setup.
    For instance, if you want to prove remotely via Bonsai, unset `RISC0_DEV_MODE` and set `BONSAI_API_URL` and `BONSAI_API_KEY` accordingly.
    Moreover, if you want to run the relay on a remote Ethereum network, you can use a different `ETH_NODE`, `ETH_CHAIN_ID` and `PRIVATE_KEY`.
    If you want to know more about the relay, you can follow this [link](https://github.com/risc0/risc0/tree/main/bonsai/ethereum-relay).

**Now you can test your deployment as follows:**

1. Send a transaction to the starter contract:

    ```bash
    cast send --private-key 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d --gas-limit 100000 0xe7f1725e7734ce288f8367e1bb143e90bb3f0512 'calculateFibonacci(uint256)' 5
    ```

2. Check the relayed result:

    ```bash
    cast call 0xe7f1725e7734ce288f8367e1bb143e90bb3f0512 'fibonacci(uint256)' 5
    ```

**Deploy a new version of your application:**

When you want to deploy a new version of the application contract, run the following command with the relay contract address noted earlier.
Set `DEPLOY_UPLOAD_IMAGES=true` if you modified your guest and need to upload a new version to Bonsai.

```bash
DEPLOY_RELAY_ADDRESS=0x5FbDB2315678afecb367f032d93F642f64180aa3 DEPLOY_UPLOAD_IMAGES=true forge script script/Deploy.s.sol --rpc-url http://localhost:8545 --broadcast
```

This will deploy only your application address and upload any updated images.
The existing relay contract and the running relay will continue to be used.

**Use the fully verifying relay:**

If you want to use Bonsai and the on-chain SNARK verifier in your local or testnet deployment, set `BONSAI_PROVING=bonsai` and rerun the steps for deployment.
When `BONSAI_PROVING=bonsai`, the zkVM guest images will be uploaded to Bonsai and the relay will use Bonsai for executing and proving the guest.
With this setup, the fully verifying relay will be used and a SNARK proof will be required to send a callback to your application.

See the [Configuring Bonsai](#Configuring Bonsai) section below for more information about using the Bonsai proving service.

### Off-chain Callback Request

The Relay exposes an HTTP REST API interface that can be used to directly send *off-chain* Callback requests to it, as an alternative to the on-chain requests.
It also provides an SDK in `rust` that can be used to interact with it. You can check out this [example](relay/examples/publish.rs).

Assuming that Anvil and the Relay are running and both the `BonsaiRelay` and `BonsaiStarter` are deployed (first 4 steps of the previous section), you can send a `Callback` request directly to the Relay by running:

```bash
cargo run --example publish 288ea9093b9000870ccd8cef93d24bba3cc5f67b14b6f9b651072e23984a379c 0xe7f1725e7734ce288f8367e1bb143e90bb3f0512 10
```

The first argument is the `image_id`, then the `BonsaiStarter` address and finally the number to compute the Fibonacci sequence.
You may need to change these values accordingly.

Once again, you can check the relayed result

```bash
cast call 0xe7f1725e7734ce288f8367e1bb143e90bb3f0512 'fibonacci(uint256)' 10
```

The Relay source code with its SDK can be found in the [risc0/risc0](https://github.com/risc0/risc0/tree/main/bonsai/ethereum-relay) github repo.

## Proving Modes

The foundry template supports two different proving modes:

1. `local` - By default, only the [executor](https://www.dev.risczero.com/docs/terminology#executor) runs your zkVM program and no proof is generated.
    Because there is no proving, this will be the fastest way to test.
    In the local, execution-only, mode a mock version of the `BonsaiRelay` will be used that does not verify proofs for callbacks.

2. `bonsai` - A proof of execution is generated by the Bonsai API.
    When using the `bonsai` proving mode, the real `BonsaiRelay` will be used and all callbacks will require a SNARK proof.

Configure your preferred mode by setting the `BONSAI_PROVING` env variable.

```bash
export BONSAI_PROVING=bonsai
```

After setting your preferred proving mode, use `forge test` to build and run your application.

### Configuring Bonsai

***Note:*** *The Bonsai proving service is still in early Alpha. To request an API key [complete the form here](https://bonsai.xyz/apply).*

To prove in `bonsai` mode, two additional environment variables are required

```bash
export BONSAI_API_KEY="YOUR_API_KEY" #see form linked above
export BONSAI_API_URL="BONSAI_URL" #provided with your api key
export BONSAI_PROVING=bonsai
forge test
```

## Project Structure

Below are the primary files in the project directory

```text
.
├── Cargo.toml                      // Definitions for cargo and rust
├── foundry.toml                    // Definitions for foundry
├── contracts                       // Your Ethereum contracts live here
│   ├── BonsaiStarter.sol           // Starter template for basic callback contract
│   └── BonsaiStarterLowLevel.sol   // Starter template for low-level callback contract
├── tests                           // Your Ethereum contract tests live here
│   ├── BonsaiStarter.t.sol         // Tests for basic callback contract
│   └── BonsaiStarterLowLevel.t.sol // Tests for low-level callback contract
└── methods                         // [zkVM guest programs] are built here
    ├── Cargo.toml
    ├── build.rs                    // Instructions for the risc0-build rust crate
    ├── guest                       // A rust crate containing your [zkVM guest programs]
    │   ├── Cargo.toml
    │   └── src
    │       └── bin                 // Your [zkVM guest programs] live here
    │           └── fibonacci.rs    // Example [guest program] for fibonacci number calculation
    └── src
        ├── main.rs                 // Glue binary for locally testing Bonsai applications
        └── lib.rs                  // Built RISC Zero guest programs are compiled into here
```

### Contracts

Ethereum contracts should be written in the `contracts` directory, where the two primary starter template contracts live.
The Solidity libraries for Bonsai can be found at [github.com/risc0/risc0](https://github.com/risc0/risc0/tree/main/bonsai/ethereum)

Contracts are built and tested with [forge], which is part of the [Foundry] toolkit.
Tests are defined in the `tests` directory.

### Methods

[RISC Zero] guest programs are defined in the `methods` directory.
This is where you will define one or more guest programs to act as a coprocessor to your on-chain logic.
More example of what you can do in the guest can be found in the [RISC Zero examples].

Code in the `methods/guest` directory will be compiled into one or more [RISC-V] binaries.
Each will have a corresponding image ID, which is a hash identifying the program.
When deploying your application, you will upload your binary to Bonsai where the guest will run when requested.
The image ID will be included in the deployment of the smart contracts to reference your guest program living in Bonsai.

Build configuration for the methods is included in `methods/build.rs`.

[Bonsai]: https://dev.bonsai.xyz/
[RISC Zero]: https://www.risczero.com/
[RISC Zero examples]: https://github.com/risc0/risc0/tree/main/examples
[RISC-V]: https://www.risczero.com/docs/reference-docs/about-risc-v
[Foundry]: https://getfoundry.sh/
[zkVM]: https://www.dev.risczero.com/terminology#zero-knowledge-virtual-machine-zkvm
[zkVM guest program]: https://www.dev.risczero.com/terminology#guest-program
