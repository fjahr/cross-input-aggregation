# Cross-Input Signature Aggregation (CISA)

CISA is a potential Bitcoin softfork that reduces transaction weight. The purpose of this repository is to collect thoughts and resources on signature aggregation schemes themselves and how they could be integrated into Bitcoin.

## Contents

- [Intro to Signature Half Aggregation](slides/2021-Q2-halfagg-impl.org)
- [Recording of Implementing Half Aggregation in libsecp256k1-zkp](https://www.youtube.com/watch?v=Dns_9jaNPNk)
- [Cross-input-aggregation savings](savings.org)
- [Integration Into The Bitcoin Protocol](#integration-into-the-bitcoin-protocol)
- [Half Aggregation And Adaptor Signatures](#half-aggregation-and-adaptor-signatures)
- [Half Aggregation And Mempool Caching](#half-aggregation-and-mempool-caching)

## Integration Into The Bitcoin Protocol

Since the verification algorithm for half and fully aggregated signatures differs from BIP 340 Schnorr Signature verification, nodes can not simply start to produce and verify aggregated signatures.
This would result in a chain split.

Taproot & Tapscript provide multiple upgrade paths:
- **Redefine `OP_SUCCESS` to `OP_CHECKAGGSIG`:**
    As pointed out in [this post to the bitcoin-dev mailing list](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2018-March/015838.html), an `OP_CHECKAGGSIG` appearing within a script that includes `OP_SUCCESS` can result in a chain split.
    That's because `OP_CHECKAGGSIG` does not actually verify the signature, but puts the public key in some datastructure against which the aggregate signature is only verified in the end - after having encountered all `OP_CHECKAGGSIG`.
    While one node sees `OP_SUCCESS OP_CHECKSIGADD`, a node with another upgrade - supposedly a softfork - may see `OP_DOPE OP_CHECKSIGADD`.
    Since they disagree how to verify the aggregate signature, they will disagree on the verification result which results in a chainsplit.
    Hence, `OP_CHECKAGGSIG` can't be used in a scripting system with `OP_SUCCESS`.
    The same argument holds for the attempt to add aggregate signatures via Tapscript's key version.
- **Define new leaf version to replace tapscript:** If the new scripting system has `OP_SUCCESS` then this does not solve the problem.
- **Define new SegWit version:**
    It is possible to define a new SegWit version that is a copy of Taproot & Tapscript with the exception that all keyspend signatures are allowed to be aggregated.
    However, keyspends can not be aggregated with other SegWit versions.

Assume that a new SegWit version is defined to deploy aggregate signatures by copying Taproot and Tapscript and allowing only keyspends to be aggregated.
This would be limiting.
For example, a spending policy `(pk(A) and pk(B)) or (pk(A) and older(N))` would usually be instantiated in Taproot by aggregating keys A and B to create a keypath spend and appending a script path for `(pk(A) and older(N))`.
It wouldn't be possible to aggregate the signature if the second spending path is used.

This [bitcoin-dev post](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2018-July/016249.html) shows that this limitation is indeed unnecessary by introducing Generalized Taproot, a.k.a. g'root  (see also [this post](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2018-October/016461.html) for a summary).
Essentially, instead of requiring that each leaf of the taproot merkle tree is a script, in g'root leafs can consist of both a public key and a script.
In order to use such a spending path, a signature for the public key must be provided, as well as the inputs to satisfy the script.
This means that the public key is moved out of the scripting system, leaving it unencumbered by `OP_SUCCESS` and other potentially dangerous Tapscript components.
Hence, signatures for these public keys _can_ be aggregated.

Consider the example policy `(pk(A) and pk(B)) or (pk(A) and older(N))` from above.
In g'root the root key `keyagg((pk(A), pk(B)))` commits via taproot tweaking to a spending condition consisting of public key `pk(A)` and script `older(N)`.
In order to spend with the latter path, the script must be satisfied and an _aggregated_ signature for `pk(A)` must exist.


## Half Aggregation And Adaptor Signatures

Half aggregation prevents using adaptor signatures ([stackexchange](https://bitcoin.stackexchange.com/questions/107196/why-does-blockwide-signature-aggregation-prevent-adaptor-signatures)).
However, a new SegWit version as outlined in section [Integration Into The Bitcoin Protocol](#integration-into-the-bitcoin-protocol) would keep signatures inside Tapscript unaggregatable.
Hence, protocols using adaptor signatures can be instantiated by having adaptor signatures only appear inside Tapscript.

This should not be any less efficient in g'root if the output can be spend directly with a script, i.e., without showing a merkle proof.
However, since this is not a normal keypath spend and explicitly unaggregatable, such a spend will stick out from other transactions.
It is an open question if this actually affects protocols built on adaptor signatures.
In other words, can such protocols can be instantiated with a Tapscript spending path for the adaptor signature but without having to use actually use that path - at least in the cooperative case?

# Half Aggregation And Mempool Caching

Nodes accepting a transaction with a half aggregate signature `(s, R_1, ..., R_n)` to their mempool would not throw it away or aggregate it with other signatures.
Instead, they keep the signature and when a block with block-wide aggregate signature `(s', R'_1, ..., R'_n')` arrives they can subtract `s` from `s'` and remove `R_1, ..., R_n`, from the block-wide aggregate signature before verifying it.
As a result, the nodes skip what they have already verified.