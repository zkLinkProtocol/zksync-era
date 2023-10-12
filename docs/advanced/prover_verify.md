# Proofs, public inputs and verification

Assuming that we have the proof for some block, let's talk about its verification. How do we know that the proof is actually proving what we want to be proven?

zkProof requires two additional parts, to make sure that it is proving the right thing:
* verification key
* public input


This way, we check that proof states that we run the correct ciruit code (thanks to verification_key), on the correct inputs (thanks to public input).

## Verification Key

You can think of the verification key as the 'hash of the circuit source code' - we described in more details in the separate article.

If you're verifying the proof, make sure that verification key is correct - preferably generate it yourself based on the source code (this might take around 30 min of computation BTW). 

If the verification key is invalid, you don't really know what you're proving (basically you'd be proving that SOME circuit was run succesfully, but you don't know which one).

That's why in zksync Era, the verification key is basically hardcoded into the [Verifier contract][verifier_sol] itself. Whenever it changes, we have to generate new solidity file, and proceed with full DiamondProxy facet upgrade.

You can see how we generate the verifier.sol using [rust here][verifier_generator]


## Public input

The other piece of the puzzle is public input. Normally you'd expect multiple public inputs coming into a single circuit, but to make things more compact, we usually accept just a single public input, which is a hash of all of them.

To be more exact, the public input to prove the batch[X] is:
```
public_input = keccak(commitment(batch[X-1]), commitment(batch[X]), other_stuff)
```

You can see the code in [verifier here][verifier_commitment_code].

This way, you know that the circuit has executed the given batch, based on the state that was the result of the execution of the previous batch.


## How can I see it on chain?

Look at this [Timelock Verifier][timelock_verifier] contract in etherscan.

## How can I verify it locally?

We created a [Era Boojum Validator CLI][era_boojum] rust tool to help you with some local testing and verification.

**WARNING:** the tool will change a little after we launch the boojum into mainnet. 

As described above, you need 3 pieces:
* a proof itself
* verification key
* public inputs


### Easier way

Run the tool: 
```shell
cargo run -- --batch 109939 --network mainnet --l1-rpc https://rpc.ankr.com/eth
```
The tool will download all 3 pieces from the Google Storage, verify the proof and compare public input information with the data received from l1 RPC.

NOTE: currently we don't compare the verification key hash with L1 (this is planned to be added once we rollout the boojum to mainnet, as only then the L1 contract is going to have this verification key).

### Harder way

Easier way would work for most of the use cases, but if you don't want to depend on the data from the internet, you can re-compute everything in more manual way. (note that you still have to download at least the proof from the internet).

You can generate the verification key, using tools in [Vk key generator][vk_key_generator].

And you can compute the public input manually - using the code in [our tests][boojum_cli_tests]


## Advanced


### Batch commitment

Ok, but what's inside the batch commitment. You can think about it as a 'hash' of the batch contents, but in practice, we generate it from 3 piieces:

```
keccak(passthrough, metadata, aux)
```

#### Passthrough data
This is the 'state' of the blockchain: combination of state root hash, and enumeration index (which is the information on how many different keys we published to L1 in the past).

For future features, we also include these values for our other shard (zkPorter), but this is not enabled in mainnet yet, so we pass 0.

So passthrough is:
```solidity
abi.encodePacked(
    _batch.indexRepeatedStorageChanges,
    _batch.newStateRoot,
    uint64(0), // index repeated storage changes in zkPorter
    bytes32(0) // zkPorter batch hash
);
```

#### Meta parameters
Meta parameters are the configuration of the blockchain itself: the code hashes that are used for the bootloader, default account, and information on which shards are enabled.


```solidity
abi.encodePacked(s.zkPorterIsAvailable, 
                 s.l2BootloaderBytecodeHash, 
                 s.l2DefaultAccountBytecodeHash);
```

#### Aux parameters

These parameters cover additional data. See the Advanced section below for more info.
```solidity
abi.encode(
    l2ToL1LogsHash,
    _stateDiffHash,
    _batch.bootloaderHeapInitialContentsHash,
    _batch.eventsQueueStateHash
)
```

### What proof system are you using 

For most of the circuits, we use STARK with FRI, but at the last step, we compress the final FRI proof (still using STARKS), and then the final final proof is being wrapped into a SNARK with KZG.

And that is the proof that is sent to L1 to be verified.





[verifier_sol]: https://github.com/matter-labs/era-contracts/blob/dev/ethereum/contracts/zksync/Verifier.sol#L286
[verifier_generator]: https://github.com/matter-labs/era-contracts/blob/dev/tools/src/main.rs
[verifier_commitment_code]: https://github.com/matter-labs/era-contracts/blob/dev/ethereum/contracts/zksync/facets/Executor.sol#L384

[era_boojum]: https://github.com/matter-labs/era-boojum-validator-cli
[vk_key_generator]: https://github.com/matter-labs/zksync-era/tree/main/prover/vk_setup_data_generator_server_fri
[boojum_cli_tests]: https://github.com/matter-labs/era-boojum-validator-cli/blob/a1b35fa514e0fbf45aa558e5caf6d0d6f2afabfa/src/main.rs#L561C7-L561C7
[timelock_verifier]: https://etherscan.io/address/0x3db52ce065f728011ac6732222270b3f2360d919
