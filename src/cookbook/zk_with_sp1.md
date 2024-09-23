# ZK proofs with SP1

Zero-knowledge proofs are an exciting new tool for decentralize applications.
Thanks to [SP1](https://github.com/succinctlabs/sp1), you can prove a Rust program with an extremely easy to use open-source library.
There are a number of other ZK proving systems both in production and under development, which can also be used inside the Kinode environment, but this tutorial will focus on SP1.

### Start

In a terminal window, start a fake node to use for development of this app.
```bash
kit boot-fake-node
```

In another terminal, create a new app using [kit](../kit/kit-dev-toolkit.md).
Use the fibonacci template, which can then be modified to calculate fibonacci numbers in a *provably correct* way.
```bash
kit new my_zk_app -t fibonacci
cd my_zk_app
kit bs
```

Take note of the basic fibonacci program in the template.
The program presents a request/response pattern where a requester asks for the nth fibonacci number, and the process calculates and returns it.
This can be seen in action by running the following command in the fake node's terminal:
```bash
m our@my_zk_app:my_zk_app:template.os -a 5 '{"Number": 10}'
```
(Change the package name to whatever you named your app + the publisher node as assigned in `metadata.json`.)

You should see a print from the process that looks like this, and a returned JSON response that the terminal prints:
```
my_zk_app: fibonacci(10) = 55; 375ns
{"Number":55}
```

### Cross-network computation

From the template, you have a program that can be used across the Kinode network to perform a certain computation.
If the template app here has the correct capabilities, other nodes will be able to message it and receive a response.
This can be seen in action by booting another fake node (while keeping the first one open) and sending the fibonacci program a message:
```
# need to set a custom name and port so as not to overlap with first node
kit boot-fake-node -p 8081 --fake-node-name fake2.os
# wait for the node to boot
m fake.os@my_zk_app:my_zk_app:template.os -a 5 '{"Number": 10}'
```
(Replace the target node ID with the first fake node, which by default is `fake.os`)

You should see `{"Number":55}` in the terminal of `fake2.os`!
This reveals a fascinating possibility: with Kinode, one can build p2p services accessible to any node on the network.
However, the current implementation of the fibonacci program is not provably correct.
The node running the program could make up a number -- without doing the work locally, there's no way to verify the result.
ZK proofs can solve this problem.

### Introducing the proof

To add ZK proofs to this simple fibonacci program, you can use the [SP1](https://github.com/succinctlabs/sp1) library to write a program in Rust, then produce proofs against it.

First, add the SP1 dependency to the `Cargo.toml` file for `my_zk_app`:
```toml
[dependencies]
...
sp1-core = { git = "https://github.com/succinctlabs/sp1.git" }
...
```

Now follow the [SP1 install steps](https://succinctlabs.github.io/sp1/getting-started/install.html) to get the tooling for constructing a provable program.
After installing you should be able to run
```
cargo prove new fibonacci
```
and navigate to a project, which conveniently contains a fibonacci function example.
Modify it slightly to match what our fibonacci program does.
You can more or less copy-and-paste the fibonacci function from your Kinode app to the `program/src/main.rs` file in the SP1 project.
It'll look like this:
```rust
#![no_main]
sp1_zkvm::entrypoint!(main);

pub fn main() {
    let n = sp1_zkvm::io::read::<u32>();
    if n == 0 {
        sp1_zkvm::io::write(&0);
        return;
    }
    let mut a: u128 = 0;
    let mut b: u128 = 1;
    let mut sum: u128;
    for _ in 1..n {
        sum = a + b;
        a = b;
        b = sum;
    }
    sp1_zkvm::io::write(&b);
}
```

Now, use SP1's `prove` tool to build the ELF that will actually be executed when the process get a fibonacci request.
Run this inside the `program` dir of the SP1 project you created:
```bash
cargo prove build
```

Next, take the generated ELF file from `program/elf/riscv32im-succinct-zkvm-elf` and copy it into the `pkg` dir of your *Kinode* app.
Go back to your Kinode app code and include this file as bytes so the process can execute it in the SP1 zkVM:
```rust
const FIB_ELF: &[u8] = include_bytes!("../../pkg/riscv32im-succinct-zkvm-elf");
```

### Building the app

Now, this app can use this circuit to not only calculate fibonacci numbers, but include a proof that the calculation was performed correctly!
The subsequent proof can be serialized and shared across the network with the result.
Take a moment to imagine the possibilities, then take a look at the full code example below:

Some of the code from the original fibonacci program is omitted for clarity, and functionality for verifying proofs our program receives from others has been added.

```rust
use kinode_process_lib::{println, *};
use serde::{Deserialize, Serialize};
use sp1_sdk::{utils, ProverClient, SP1ProofWithPublicValues, SP1Stdin};

/// The ELF we want to execute inside the zkVM.
const FIB_ELF: &[u8] = include_bytes!("../../pkg/riscv32im-succinct-zkvm-elf");

wit_bindgen::generate!({
    path: "target/wit",
    world: "my-zk-app-template-dot-os-v0",
    generate_unused_types: true,
    additional_derives: [serde::Deserialize, serde::Serialize, process_macros::SerdeJsonInto],
});

#[derive(Debug, Serialize, Deserialize)]
enum FibonacciRequest {
    /// Send this locally to ask a peer for a proof
    ProveIt { target: NodeId, n: u32 },
    /// Send this to a peer's fibonacci program
    Number(u32),
}

#[derive(Debug, Serialize, Deserialize)]
enum FibonacciResponse {
    /// What we return to the local request
    Proven(u128),
    /// What we get from a remote peer
    Proof, // bytes in message blob
}

/// PROVE the nth Fibonacci number
/// since we are using u128, the maximum number
/// we can calculate is the 186th Fibonacci number
/// return the serialized proof
fn fibonacci_proof(n: u32) -> Vec<u8> {
    let mut stdin = SP1Stdin::new();
    stdin.write(&n);

    let client = ProverClient::new();
    let (pk, _) = client.setup(FIB_ELF);
    let proof = client.prove(&pk, stdin).run().unwrap();

    println!("successfully generated and verified proof for fib({n})!");
    serde_json::to_vec(&proof).unwrap()
}

fn handle_message(our: &Address) -> anyhow::Result<()> {
    let message = await_message()?;

    match serde_json::from_slice(message.body())? {
        FibonacciRequest::ProveIt { target, n } => {
            if message.source().node() != our.node() {
                return Err(anyhow::anyhow!("got a request from a non-local node!"));
            }

            let res = Request::to(Address::new(
                target,
                (our.process(), our.package(), our.publisher()),
            ))
            .body(serde_json::to_vec(&FibonacciRequest::Number(n))?)
            .send_and_await_response(30)??;

            let Ok(FibonacciResponse::Proof) = serde_json::from_slice(res.body()) else {
                return Err(anyhow::anyhow!("got a bad response!"));
            };

            let proof_bytes = res
                .blob()
                .ok_or_else(|| anyhow::anyhow!("no proof in response"))?
                .bytes;

            // Deserialize and verify the proof
            let client = ProverClient::new();
            let (_, vk) = client.setup(FIB_ELF);
            let mut proof: SP1ProofWithPublicValues = serde_json::from_slice(&proof_bytes)?;
            client.verify(&proof, &vk).map_err(|e| anyhow::anyhow!("{e:?}"))?;

            // Read result from proof
            let _ = proof.public_values.read::<u32>();
            let _ = proof.public_values.read::<u32>();
            let output = proof.public_values.read::<u128>();

            Response::new()
                .body(serde_json::to_vec(&FibonacciResponse::Proven(output))?)
                .send()?;
        }
        FibonacciRequest::Number(n) => {
            let proof = fibonacci_proof(n);
            Response::new()
                .body(serde_json::to_vec(&FibonacciResponse::Proof)?)
                .blob_bytes(proof)
                .send()?;
        }
    }
    Ok(())
}

call_init!(init);

fn init(our: Address) {
    utils::setup_logger();
    println!("fibonacci: begin");
    loop {
        match handle_message(&our) {
            Ok(()) => {}
            Err(e) => {
                println!("fibonacci: error: {:?}", e);
            }
        };
    }
}
```

### Test it out

Install this app on two nodes -- they can be the fake `kit` nodes from before, or real ones on the network.
Next, send a message from one to the other, asking it to generate a fibonacci proof!
```
m our@my_zk_app:my_zk_app:template.os -a 30 '{"ProveIt": {"target": "fake.os", "n": 10}}'
```
As usual, set the process ID to what you used, and set the `target` JSON value to the other node's name.
Try a few different numbers -- see if you can generate a timeout (it's set at 30 seconds now, both in the terminal command and inside the app code).
If so, the power of this proof system is demonstrated: a user with little compute can ask a peer to do some work for them and quickly verify it!

In just over 100 lines of code, you have written a program that can create, share across the network, and verify ZK proofs.
Use this as a blueprint for similar programs to get started using ZK proofs in a brand new p2p environment!
