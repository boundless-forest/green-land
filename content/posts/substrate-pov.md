---
title: "The polkadot parachain PoV"
date: 2023-05-15
tags: 
    - tech
---

# Introduction

If you know something about the Polkadot Parachain, you must have heard of one term: `PoV`. It appears in the substrate pull requests, the polkadot official docs, forum and almost everywhere the Polkadot Parachain appears. So, what is the PoV? Why does it appear so often in the ecosystem? I'll try to dig into a little deeper.

# What is the PoV?

Once our question was clear, I easily found this helpful answer [What does PoV stand for?](https://substrate.stackexchange.com/questions/518/what-does-pov-stand-for), but this answer does not show what is in the PoV? just the storage trie proof? anything else? I guess it's a bunch of encoded bytes, but what is the raw data structure?

The PoV structure definition:

```rs
/// A Proof-of-Validity
pub struct PoV {
	/// The block witness data.
	pub block_data: BlockData,
}

/// Parachain block data.
///
/// Contains everything required to validate para-block, may contain block and witness data.
pub struct BlockData(#[cfg_attr(feature = "std", serde(with = "bytes"))] pub Vec<u8>);
```

Right, it's a bunch of bytes indeed, the next step is to dig out the raw data:

```rs
// The parachain block that is created by a collator.
///
/// This is send as PoV (proof of validity block) to the relay-chain validators. There it will be
/// passed to the parachain validation Wasm blob to be validated.
#[derive(codec::Encode, codec::Decode, Clone)]
pub struct ParachainBlockData<B: BlockT> {
	/// The header of the parachain block.
	header: B::Header,
	/// The extrinsics of the parachain block.
	extrinsics: sp_std::vec::Vec<B::Extrinsic>,
	/// The data that is required to emulate the storage accesses executed by all extrinsics.
	storage_proof: sp_trie::CompactProof,
}
```

Most interestingly, the PoV includes not only the storage trie proof but also the header and extrinsic. On second thought, this makes sense. The PoV is much more a generic term to describe the thing relay chain needed to verify the parachain candidate blocks, the block header and extrinsics are obviously part of it.

# PoV Size Limit

According to the [What does PoV stand for?](https://substrate.stackexchange.com/questions/518/what-does-pov-stand-for):

> The PoV has a maximum limit for parachain blocks. On Polkadot it is currently configured to 5 MB so that there are limits to how long it could take to gossip that information to the relevant Polkadot validators.

The Polkadot Parachain `max_pov_size` is now 5M. So, I have two questions:

1. Why it cannot be set to be a larger value, for example, e.g. 10M?
2. What happens if the parachain pov size over 5M?

The chatGPT answer:

1. The Proof-of-Validity (PoV) block size in Polkadot's parachain is set to a certain limit (e.g., 5M) due to several reasons:
    - Scalability: One of the primary goals of Polkadot is to create a scalable network. Larger PoV blocks would require more time to process and validate, which could lead to increased latency and reduced throughput on the network.
    - Decentralization: A larger PoV block size would increase the resource requirements for validators, making it more difficult for smaller validators to participate in the network. This could lead to a more centralized network, which goes against the core principles of Polkadot.
    - Security: Large PoV blocks could make the network more susceptible to spam attacks, where an attacker could create a large number of transactions to congest the network and cause delays.

2. If a parachain PoV size exceeds the 5M limit:
    - The block would be deemed invalid, and validators would not include it in the relay chain.
    - The parachain would need to split the transactions into smaller blocks, adhering to the 5M size limit, and submit them separately for validation.

# PoV Creation And Verification

![PoV Creation And Verification](/pov-creation-verification.png)

When a parachain collator is set up and running, it will collect and validate the extrinsics of the network and pack these legitimation extrinsics into a block, just as the standalone substrate chain validator does. In addition, the parachain collator requires a further step to collect the `storage_proof`. The whole process is done in the `start_collator() -> produce_candidate()`, you can dig it deeper if you are interested.

Once the `ParachainBlockData` has been created by the callator, the next step is wrap the data into the `CollationGenerationMessage` message, which is one of the message types in the Relay Chain [Overseer](https://paritytech.github.io/polkadot/book/node/overseer.html) protocol. Once the messages are received in the Relay Chain, the validators who responsible for validating this parachain candidate are online and ready to go. There are two other message types involved in triggering this validation, namely `ValidateFromChainState` or `ValidateFromExhaustive`. 

```rs
/// Does basic checks of a candidate. Provide the encoded PoV-block. Returns `Ok` if basic checks
/// are passed, `Err` otherwise.
fn perform_basic_checks(
	candidate: &CandidateDescriptor,
	max_pov_size: u32,
	pov: &PoV,
	validation_code_hash: &ValidationCodeHash,
) -> Result<(), InvalidCandidate> {
	let pov_hash = pov.hash();

	let encoded_pov_size = pov.encoded_size();
	if encoded_pov_size > max_pov_size as usize {      // here
		return Err(InvalidCandidate::ParamsTooLarge(encoded_pov_size as u64))
	}

	if pov_hash != candidate.pov_hash {
		return Err(InvalidCandidate::PoVHashMismatch)
	}

	if *validation_code_hash != candidate.validation_code_hash {
		return Err(InvalidCandidate::CodeHashMismatch)
	}

	if let Err(()) = candidate.check_collator_signature() {
		return Err(InvalidCandidate::BadSignature)
	}

	Ok(())
}
```

If the `perform_basic_checks` failed, then this parachain candidate will not to be included in the Relay Chain, and your parachain will hang. That's why the size of the parachain block candidate is important.

# The size of your chain PoV?

Once the block is included in the Relay Chain and finalised, the PoV is no longer required and is not stored indefinitely by the validators. Therefore, it's not currently possible to monitor the PoV size of your parachain using the Polkadot Apps or SubScan. The only clue you can get now is from the collator logs, something like this:

```sh
2022-03-21 14:48:06 [Parachain] PoV size { header: 0.181640625kb, extrinsics: 3.525390625kb, storage_proof: 4.6123046875kb }
2022-03-21 14:48:06 [Parachain] Compressed PoV size: 7.3828125kb
2022-03-21 14:48:06 [Parachain] Produced proof-of-validity candidate. block_hash=0x4f1e697ca979e8f00015ba04f544cb87a31aaf22867a4dd0c36c4a2a69349e09
```