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