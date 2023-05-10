---
icon: workflow
order: 7
---
#### verifying with the EVM ◊

Note that the above prove and verify stats can also be run with an EVM verifier. This can be done by generating a verifier smart contract after generating the proof

```bash
# gen proof
ezkl --bits=16 -K=17 prove -D ./examples/onnx/1l_relu/input.json -M ./examples/onnx/1l_relu/network.onnx --proof-path 1l_relu.pf --vk-path 1l_relu.vk --params-path=kzg.params --transcript=evm
```
```bash
# gen evm verifier
ezkl -K=17 --bits=16 create-evm-verifier -D ./examples/onnx/1l_relu/input.json -M ./examples/onnx/1l_relu/network.onnx --deployment-code-path 1l_relu.code --params-path=kzg.params --vk-path 1l_relu.vk --sol-code-path 1l_relu.sol
```
```bash
# Verify (EVM)
ezkl -K=17 --bits=16 verify-evm --proof-path 1l_relu.pf --deployment-code-path 1l_relu.code
```

Note that the `.sol` file above can be deployed and composed with other Solidity contracts, via a `verify()` function. Please read [this document](https://hackmd.io/QOHOPeryRsOraO7FUnG-tg) for more information about the interface of the contract, how to obtain the data needed for its function parameters, and its limitations.

The above pipeline can also be run using [proof aggregation](https://ethresear.ch/t/leveraging-snark-proof-aggregation-to-achieve-large-scale-pbft-based-consensus/11588) to reduce proof size and verifying times, so as to be more suitable for EVM deployment. A sample pipeline for doing so would be:

```bash
# Generate a new 2^20 SRS
ezkl -K=20 gen-srs --params-path=kzg.params
```

```bash
# Single proof -> single proof we are going to feed into aggregation circuit. (Mock)-verifies + verifies natively as sanity check
ezkl -K=17 --bits=16 prove --transcript=poseidon --strategy=accum -D ./examples/onnx/1l_relu/input.json -M ./examples/onnx/1l_relu/network.onnx --proof-path 1l_relu.pf --params-path=kzg.params  --vk-path=1l_relu.vk
```

```bash
# Aggregate -> generates aggregate proof and also (mock)-verifies + verifies natively as sanity check
ezkl -K=20 --bits=16 aggregate --app-logrows=17 --transcript=evm -M ./examples/onnx/1l_relu/network.onnx --aggregation-snarks=1l_relu.pf --aggregation-vk-paths 1l_relu.vk --vk-path aggr_1l_relu.vk --proof-path aggr_1l_relu.pf --params-path=kzg.params
```

```bash
# Generate verifier code -> create the EVM verifier code
ezkl -K=17 --bits=16 create-evm-verifier-aggr --deployment-code-path aggr_1l_relu.code --params-path=kzg.params --vk-path aggr_1l_relu.vk
```

```bash
# Verify (EVM) ->
ezkl -K=17 --bits=16 verify-evm --proof-path aggr_1l_relu.pf --deployment-code-path aggr_1l_relu.code
```

Also note that this may require a local [solc](https://docs.soliditylang.org/en/v0.8.17/installing-solidity.html) installation, and that aggregated proof verification in Solidity is not currently supported.

For both pipelines the resulting verifier can be deployed to an EVM instance (mainnet or otherwise !) using the `deploy-verifier-evm` command:

```bash
Deploys an EVM verifier

Usage: ezkl deploy-verifier-evm [OPTIONS] --secret <SECRET> --rpc-url <RPC_URL>

Options:
  -S, --secret <SECRET>
          The path to the wallet mnemonic
  -U, --rpc-url <RPC_URL>
          RPC Url
      --deployment-code-path <DEPLOYMENT_CODE_PATH>
          The path to the desired EVM bytecode file (optional), either set this or sol_code_path
      --sol-code-path <SOL_CODE_PATH>
          The path to output the Solidity code (optional) supercedes deployment_code_path in priority
  -h, --help
          Print help

```

For instance:

```bash
ezkl deploy-verifier-evm -S ./mymnemonic.txt -U myethnode.xyz --deployment-code-path aggr_1l_relu.code
```

You can also send proofs to be verified on deployed contracts using `send-proof`:

```bash
Send a proof to be verified to an already deployed verifier

Usage: ezkl send-proof-evm --secret <SECRET> --rpc-url <RPC_URL> --addr <ADDR> --proof-path <PROOF_PATH>

Options:
  -S, --secret <SECRET>          The path to the wallet mnemonic
  -U, --rpc-url <RPC_URL>        RPC Url
      --addr <ADDR>              The deployed verifier address
      --proof-path <PROOF_PATH>  The path to the proof
  -h, --help                     Print help

```

For instance:

```bash
ezkl send-proof-evm -S ./mymnemonic.txt -U myethnode.xyz --addr 0xFFFF --proof-path my.snark
```