---
title: Getting Ready to Transition to Post-Quantum Security in XRPL

---

---
layout: post
title:  "Getting Ready to Transition to Post-Quantum Security in XRPL"
date:   2025-08-20 15:18:33 -0400
categories: post
---

# Getting Ready to Transition to Post-Quantum Security in XRPL

During the [XRPL Core Developer Boot Camp 2025](https://www.xrpl-commons.org/build/core-dev-bootcamp) we had an opportunity to discuss about how the migration to post-quantum security should happen in XRPL. This discussion reminded me that a month ago, while I was listening to the [365th episode](https://zeroknowledge.fm/podcast/365/) of the [Zero Knowledge Podcast](https://zeroknowledge.fm/), the guest, [Kostas Kryptos](https://x.com/kostascrypto/) from [Mysten Labs](https://www.mystenlabs.com/), talked about his group's brilliant idea about a backward compatible technique for transitioning to post-quantum security on the **Q-day**, the day powerful enough quantum-computers are available and can be used to break widely used cryptosystems such as ECDSA, EdDSA, RSA, etc. I explained this idea with some of the fellows in the boot camp, however I could not find any document explaining this idea on the internet (see [my X post](https://x.com/_taghi/status/1947191621759447044)). Few days after the boot camp concluded and I came back to Canada I tried to dig out more about the specification of this idea and happily they published the details in [(Baldimtsi et. al., 2025)](https://eprint.iacr.org/2025/1368) only a few days after I came back.

I found it a good opportunity to see whether and how this idea can be applied to `rippled`, the XRPL core software, as I got very good insights about the source code during the boot camp. In this post, I first explain the idea proposed by Mysten Labs and then discuss regarding the XRPL readiness to use this idea. 

![Firefly AI Generated Image](https://hackmd.io/_uploads/Hy_WNwmKeg.jpg)



The backward compatibility property of the Mysten Labs' idea ensures that the security of the existing blockchain addresses are preserved if these addresses are derived from an **EdDSA** (e.g., Ed25519) public key. This technique does not require to enforce any updates to the key derivation mechanism in wallets or any forks in the blockchain. This sounds very surprising as it is a well-known fact that the security of EdDSA is based on the computationally intractability assumption of the *Elliptic Curve Discrete Logarithm Problem (ECDLP)*, which can be broken using Shor's algorithm [(Shor, 1994)](https://doi.org/10.1109/SFCS.1994.365700) on quantum computers. It is worth emphasizing that the main concern in this document, as well as in Mysten Labs’ proposal, is the issue of *“sleeping accounts”* [(Baldimtsi et al., 2025)](https://eprint.iacr.org/2025/1368): accounts whose owners are unlikely to migrate their funds to post-quantum (PQ) accounts before Q-day.


## Preliminaries
Before presenting the idea, let me briefly review some preliminaries, including a comparison of the standard approaches to key derivation in ECDSA and EdDSA.

### ECDSA Key Pair Generation
In the ECDSA key generation, the secret key ($sk$) is randomly sampled from the interval $[1, q – 1]$, where $q$ is a sufficiently large prime number. Then, the public key $pk$ is derived directly as
$$
pk = sk\cdot B,
$$ where $B$ is the base point. The parameters $q$ and $B$ are specified according to the  parameters of the elliptic curve used. For example, [XRPL uses the famous](https://xrpl.org/docs/concepts/accounts/cryptographic-keys#secp256k1-key-derivation) **secp256k1** parameters specified by Certicom Corp. in [(Brown, 2010)](https://www.secg.org/sec2-v2.pdf). These parameters became more popular after they were [used in Bitcoin](https://en.bitcoin.it/wiki/Secp256k1). 


### EdDSA Key Pair Generation
The Mysten Labs' idea is based on EdDSA's standard for key generation, and it is different to ECDSA. [RFC 8032](https://datatracker.ietf.org/doc/html/rfc8032) ([Section 5.1.5](https://datatracker.ietf.org/doc/html/rfc8032#section-5.1.5)) describes the key generation of **Ed25519**, a famous parameter set of the EdDSA algorithm, which [XRPL aslo supports it](https://xrpl.org/docs/concepts/accounts/cryptographic-keys#ed25519-key-derivation). According to this standard, the secret key ($sk$) is a 32-byte (256 bits) randomly sampled number, and the public key ($pk$) is derived as 
$$
\begin{aligned}
ss &= \text{HashToScalar}(\textsf{SHA512}(sk)[0:32])\\
pk &= ss\cdot B,
\end{aligned}
$$ where $[0:32]$ denotes the lower 32 bytes of the $\textsf{SHA512}$'s output. This output truncation of $\textsf{SHA512}$ is referred to as "`SHA-512Half`" in [XRPL's documents](https://xrpl.org/docs/references/protocol/data-types/basic-data-types#hashes). Moreover, the $\text{HashtoScalar}$ procedure prunes (clamps) the 32-bytes result to derive a *secret scalar* $ss$ which is suitable for Ed25519 (The procedure and the rationale behind this is [described by Neil Madden](https://neilmadden.blog/2020/05/28/whats-the-curve25519-clamping-all-about/)). Finally, the derived $ss$ is multiplied by $B$, where $B$ denotes the base point of Ed25519.

### Elliptic Curve Discrete Logarithm Problem (ECDLP)
Roughly speaking, the ECDLP is defined as finding the scalar $s$ given $B$, denoting the base point, and a point $P$ in an elliptic curve, such that
$$
P = s \cdot B.
$$ECDLP is *apparently* intractable on classical computers if the elliptic curve parameters are carefully chosen. This problem is the basis for the security of ECDSA and EdDSA: given a public key $pk$ in either scheme, it is computationally infeasible to recover the corresponding private key $sk$ in ECDSA or secret scalar $ss$ in EdDSA.

However, Peter W. Shor first presented an algorithm in 1994 [(Shor, 1994)](https://doi.org/10.1109/SFCS.1994.365700) that can solve the ECDLP in polynomial time on quantum computers.

### Zero-knowledge Succinct Non-interactive Argument of Knowledge (zkSNARK)
Mysten Labs' method uses the idea of zkSNARKs. Here, I’ll briefly and informally explain what they are. A zkSNARK is a family of cryptographic protocols involving a prover and a verifier. The prover can generate a succinct, non-interactive argument showing they know a witness to an $\mathcal{NP}$ problem, without revealing the witness itself. The verifier, on the other hand, can check the validity of this argument without learning anything about the underlying witness.

Let me illustrate this with a simple example: suppose the prover knows the pre-image of a hash. That is, they know some value $w$ (witness) such that $$h = \mathcal{H}(w).$$ Using a zkSNARK protocol, the prover constructs a proof $\pi$ attesting to their knowledge of $w$, and sends $\pi$ along with $h$ (public input) to the verifier. The verifier then runs the verification algorithm and becomes convinced that the prover does know the pre-image of $h$.

The security of zkSNARKs against a malicious prover relies on a cryptographic hard problem, such as the aforementioned ECDLP, collision-resistance hash function (CRH), square span program (SSP), etc. Based on the underlying assumption, zkSNARKs can be categorized as either post-quantum secure or quantum-vulnerable.



# Post-Quantum Readiness

## Post-Quantum Readiness of EdDSA (Mysten Labs Idea)
While Shor's algorithm can solve ECDLP on quantum computers, it is assumed that many hash functions, such as $\textsf{SHA512}$, remain secure on Q-day. Consequently, even if a quantum adversary derives $ss$ from a EdDSA public key $pk$, it is still unable to recover $sk$. This let the real owner of the wallet be able to provide an **argument-of-ownership** (i.e., proof-of-ownership) of their keys, using zkSNARK protocols, even whether $ss$ is exposed or not. In other words, even if quantum adversaries break ECDLP and recover $ss$ from the public key $pk$, they still cannot determine the original secret key $sk$, which is the pre-image of $ss$. This allows the legitimate owner of an EdDSA-based account to generate a zero-knowledge argument of this knowledge, thereby proving possession of the account. This argument must be generated through **post-quantum secure zkSNARKs**, e.g., [STARK](https://eprint.iacr.org/2018/046), [Aurora](https://eprint.iacr.org/2018/828), [Ligero](https://eprint.iacr.org/2022/1608), etc.

There are multiple possible approaches to implementing this using existing zkSNARKs. Fortunately, since no changes to the current EdDSA are required, we likely have sufficient time to determine how to construct and verify this argument-of-ownership by Q-day. This argument can be used to transfer the funds from EdDSA accounts. Mysten Labs suggested an NP relation of $pk$ (public input) and $sk$ (private input or witness). However, by assuming that by Q-day $ss$ can be recovered from $pk$, a simpler relation, witch can be instantiated by proof of pre-image of $\textsf{SHA512}$ more efficiently.

## Post-Quantum Readiness of ECDSA
As we earlier discussed, $pk$ is directly derived from $sk$ in ECDSA. On Q-day, the quantum adversaries can derive the $sk$ from $pk$ and they will have the same knowledge as the real account owners (despite the EdDSA where the quantum adversary is able to recover $ss$ not $sk$). In this case, it is impossible for validators to distinguish the real ECDSA based account owners from quantum adversaries. Despite this problem existing in many blockchains, several communities recognized it long ago. 

### Bitcoin
The Bitcoin community recommended [avoiding key reuse](https://bitcoin.stackexchange.com/questions/118919/address-reuse-and-ecdsa-concern) and using unique addresses for receiving payments. In P2PKH and P2SH addresses, the ECDSA public key remains hidden (hashed) until the first time the funds are spent. Consequently, if the public key $pk$ has not yet been revealed, a quantum computer need to compute the pre-image of 160-bit hash 
$$\textsf{RIPEMD160}(\textsf{SHA256}(pk)),$$ to obtain $pk$ and, from it, derive the corresponding private key $sk$ using Shor's algorithm. Computing the pre-image ($pk$) would require approximately $2^{80}$ evaluations, using Grover’s algorithm ([Grover,1996](https://dl.acm.org/doi/10.1145/237814.237866)),  from P2PKH or P2SH addresses, i.e, the provided security level is 80 bits. While [some have raised concerns](https://x.com/myaksetig/status/1920918103883501827) about this relatively low security level under current cryptanalysis techniques, [NIST notes](https://csrc.nist.gov/csrc/media/Events/2024/fifth-pqc-standardization-conference/documents/papers/on-practical-cost-of-grover.pdf) that even for AES-128, which has a theoretical Grover cost of $2^{64}$, the practical impact is limited due to Grover’s poor parallelization properties. 

Despite this, it is worth recalling that Bitcoin addresses with revealed public keys remain vulnerable to quantum attacks. For example, [8.7% of all bitcoins](https://research.mempool.space/utxo-set-report/) are stored in the early Bitcoin address type, P2PK, where public keys are directly visible. Additionally, funds are also held in other address types where the public key has been exposed due to address reuse. [Deloitte reports that](https://www.deloitte.com/nl/en/services/consulting-risk/perspectives/quantum-computers-and-the-bitcoin-blockchain.html) 25% of all bitcoins are stored in public-key exposed addresses.

### Ethereum

Ethereum is almost similar, as it also uses ECDSA implemented in [`libsecp256k1`](https://github.com/bitcoin-core/secp256k1)  maintained by Bitcoin. However, Ethereum has one advantage and one disadvantage compared to Bitcoin. The advantage of Ethereum is that it computes $\textsf{Keccak256}$ of the public key to generate address, which provides a higher security level against Grover algorithm for the [scenario that the $pk$ is not revealed](https://ethresear.ch/t/how-to-hard-fork-to-save-most-users-funds-in-a-quantum-emergency/18901) through sending a transaction. The disadvantage is that Ethereum is an **account model** blockchain, where accounts prefer to reuse the same address. [Deloitte reports that](https://www.deloitte.com/nl/en/services/consulting-risk/perspectives/quantum-risk-to-the-ethereum-blockchain.html) 65% of all ether are stored in public-key exposed addresses.


### HD and Mnemonic Wallets
The Hierarchical deterministic (HD) wallet, explained in [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki), employs a system for deriving a tree of keypairs from a single seed to build a wallet structure on top of such a tree. The derivation of the master key and child keys in HD wallet is done through $\textsf{HMAC-SHA512}$. The presence of hash function in the key derivation lets us again use the argument-of-knowledge of the pre-image of a hash on Q-day, the knowledge quantum computers cannot achieve by breaking quantum-vulnerable cryptographic hard problems.

Additionally, in mnemonic wallets, explained in [BIP39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki), a secret binary seed is generated from a mnemonic sentence given to [PBKDF2](https://datatracker.ietf.org/doc/html/rfc2898#section-5.2), where the pseudorandom function is $\textsf{HMAC-SHA512}$ and the iteration count is 2048. Here, again the presence of hash function helps ECDSA based accounts to be able to generate argument-of-knowledge of the pre-image on Q-day. 

Vitalik, the founder of Ethereum, explains [here](https://ethresear.ch/t/how-to-hard-fork-to-save-most-users-funds-in-a-quantum-emergency/18901) how the BIP32 method of key generation can be employed to recover vulnerable accounts from a quantum emergency.

Furthermore, [EIP-7693](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki) presented a technique to efficiently generate the argument-of-knowledge of the secret pre-image in BIP32 or the mnemonic sentence in BIP39 on Q-day.

Both Bitcoin and Ethereum commonly rely on BIP32 together with BIP39. However, from the address alone, it is impossible to determine whether the underlying key was derived through BIP32/BIP39 or generated directly. Consequently, one cannot identify what fraction of bitcoins or ether is held in addresses originating from such derivation schemes.

## Post-Quantum Readiness of XRP Ledger
After this brief background, I can explain the quantum readiness of accounts on the XRP Ledger more clearly. I only need to investigate whether hash functions are used in the key derivation process or not. In this section, I will refer to specific lines of code or documentation whenever possible.


XRPL supports both **ECDSA** and **EdDSA**.  
- **ECDSA** is implemented over the `secp256k1` curve using [`libsecp256k1`](https://github.com/bitcoin-core/secp256k1), and  
- **EdDSA** is implemented over the `ed25519` curve using [`ed25519-donna`](https://github.com/floodyberry/ed25519-donna).  

As the Mysten Labs suggested (explained earlier), the EdDSA is quantum ready due to its standard key derivation mechanism. Some XRPL client libraries such as [`xrpl-py`](https://xrpl-py.readthedocs.io/en/stable/source/xrpl.wallet.html#xrpl.wallet.Wallet.create) and [`xrpl.js`](https://github.com/XRPLF/xrpl.js/blob/de28f4057e9ba3faa478957955d428dd89d2a5da/packages/xrpl/src/Wallet/index.ts#L36) use EdDSA by default. However, [**`rippled`**](https://github.com/XRPLF/rippled) uses ECDSA *by default* for [creating a new wallet](https://github.com/XRPLF/rippled/blob/b04d239926f265f2c7b5e1fd4a86b76d2479f322/src/xrpld/rpc/handlers/WalletPropose.cpp#L134) through RPC call (EdDSA can be still chosen). Also, `rippled` uses ECDSA to generate keypairs for [nodes](https://github.com/XRPLF/rippled/blob/b04d239926f265f2c7b5e1fd4a86b76d2479f322/src/xrpld/app/main/NodeIdentity.cpp#L54) and [validators](https://github.com/XRPLF/rippled/blob/b04d239926f265f2c7b5e1fd4a86b76d2479f322/src/xrpld/app/misc/detail/ValidatorKeys.cpp#L47). It worth noting that nodes and validators use their keys actively for [peer-to-peer authentication](https://xrpl.org/docs/concepts/networks-and-servers/peer-protocol#node-key-pair) and [validation messages](https://xrpl.org/docs/concepts/consensus-protocol/consensus-structure#compare-results), respectively; and not for keeping or transferring funds on the ledger. Therefore, We can assume that their algorithms will be changed to PQC algorithms when the quantum threats become more realistic. 

Alternatively, it is worth recalling that the main concern lies with *sleeping accounts*, those that keep funds on the ledger and are *unlikely* to migrate them to PQ accounts before Q-day. Determining the ratio of accounts on the XRP Ledger that use ECDSA versus EdDSA would be a valuable direction for future analysis.


As mentioned before, the post-quantum readiness of ECDSA accounts on XRPL depends on whether their key generation mechanism include hash functions, as in BIP39 and BIP39. **The good news** is that XRPL provides functionalities to generate new secret keys deterministically from a seed, which have been there since the beginning of XRPL. Even its genesis account, appeared in the first ledger, is generated from the seed value [`masterpassphrase`](https://github.com/XRPLF/rippled/blob/b04d239926f265f2c7b5e1fd4a86b76d2479f322/src/xrpld/app/ledger/Ledger.cpp#L184) from very beginning through calling [`generateKeyPair(KeyType type, Seed const& seed)`](https://github.com/XRPLF/rippled/blob/b04d239926f265f2c7b5e1fd4a86b76d2479f322/src/libxrpl/protocol/SecretKey.cpp#L369) method. 

Through the `generateKeyPair` method, if the chosen algorithm is ECDSA, the function [`deriveDeterministicRootKey(Seed const& seed)`](https://github.com/XRPLF/rippled/blob/afc05659edbc22d04b13402f157832778556f7b7/src/libxrpl/protocol/SecretKey.cpp#L86) computes the `SHA-512Half` of the given `seed` to derive the ECDSA root secret key. The `generateKeyPair` method is used as **the standard method** for generating a new (deterministic) wallet in [`walletPropose(Json::Value const& params)`](https://github.com/XRPLF/rippled/blob/b04d239926f265f2c7b5e1fd4a86b76d2479f322/src/xrpld/rpc/handlers/WalletPropose.cpp), which is called from the [`"wallet_propose"`](https://github.com/XRPLF/rippled/blob/b04d239926f265f2c7b5e1fd4a86b76d2479f322/src/xrpld/rpc/detail/Handler.cpp#L192) RPC call. The secret `seed` is either provided as a parameter to this RPC call or generated by `rippled` and returned. 

Whenever a client wants to sign a transaction, the seed is required (rather than, for instance, directly supplying an ECDSA secret key). From this seed, both the secret and public keys are derived. More specifically, in  `rippled`, the [`keypairForSignature`](https://github.com/XRPLF/rippled/blob/b04d239926f265f2c7b5e1fd4a86b76d2479f322/src/xrpld/rpc/detail/RPCHelpers.cpp#L795) function gets `seed` through an [RPC sign request](https://xrpl.org/docs/references/http-websocket-apis/admin-api-methods/signing-methods/sign) and re-generates keypairs through passing the `seed` to [`generateKeyPair`](https://github.com/XRPLF/rippled/blob/b04d239926f265f2c7b5e1fd4a86b76d2479f322/src/xrpld/rpc/detail/RPCHelpers.cpp#L927C5-L927C11). 


In summary, XRPL's support for both EdDSA and ECDSA allows EdDSA-based accounts to be considered post-quantum ready, as their standard employs $\textsf{SHA512}$ for secret key derivation. Moreover, XRPL's native standard for deriving ECDSA keys from a `seed`, from very beginning, also using the same hash function, **provides post-quantum readiness for all ECDSA-based accounts as well**. I also reviewed the `rippled` codebase to confirm that users have to comply with the XRPL standard. They are required to provide the `seed` when signing transactions, rather than supplying only the ECDSA secret key. Accordingly, it is reasonable to assume that all XRPL account owners preserve the `seed` corresponding to their account. Therefore, on Q-day, they would be able to provide a post-quantum secure argument-of-knowledge to prove real ownership of their account, as quantum computers may not be able to recover the `seed` due to the post-quantum resistance of hash functions (e.g., $\textsf{SHA512}$).

![Gemini_Generated_Image_gritzzgritzzgrit](https://hackmd.io/_uploads/H135HwXFee.jpg)



# Conclusion

In this post, I examined the quantum-readiness of the XRPL ledger based on its native account creation standard, which relies on **hash functions**. This property provides reassurance for Q-day, as real account owners will still be able to prove their ownership through a quantum-resistant argument-of-knowledge.  

Building on this readiness, the next step is to design a secure and efficient mechanism for generating such arguments of knowledge. With current technologies, post-quantum secure proof systems such as STARKs or Aurora zkSNARKs may serve as practical candidates for this purpose.

---
layout: post
title:  "Getting Ready to Transition to Post-Quantum Security in XRPL"
date:   2025-08-20 15:18:33 -0400
categories: post
---

# Getting Ready to Transition to Post-Quantum Security in XRPL

During the [XRPL Core Developer Boot Camp 2025](https://www.xrpl-commons.org/build/core-dev-bootcamp) we had an opportunity to discuss about how the migration to post-quantum security should happen in XRPL. This discussion reminded me that a month ago, while I was listening to the [365th episode](https://zeroknowledge.fm/podcast/365/) of the [Zero Knowledge Podcast](https://zeroknowledge.fm/), the guest, [Kostas Kryptos](https://x.com/kostascrypto/) from [Mysten Labs](https://www.mystenlabs.com/), talked about his group's brilliant idea about a backward compatible technique for transitioning to post-quantum security on the **Q-day**, the day powerful enough quantum-computers are available and can be used to break widely used cryptosystems such as ECDSA, EdDSA, RSA, etc. I explained this idea with some of the fellows in the boot camp, however I could not find any document explaining this idea on the internet (see [my X post](https://x.com/_taghi/status/1947191621759447044)). Few days after the boot camp concluded and I came back to Canada I tried to dig out more about the specification of this idea and happily they published the details in [(Baldimtsi et. al., 2025)](https://eprint.iacr.org/2025/1368) only a few days after I came back.

I found it a good opportunity to see whether and how this idea can be applied to `rippled`, the XRPL core software, as I got very good insights about the source code during the boot camp. In this post, I first explain the idea proposed by Mysten Labs and then discuss regarding the XRPL readiness to use this idea. 

![Firefly AI Generated Image](https://hackmd.io/_uploads/Hy_WNwmKeg.jpg)



The backward compatibility property of the Mysten Labs' idea ensures that the security of the existing blockchain addresses are preserved if these addresses are derived from an **EdDSA** (e.g., Ed25519) public key. This technique does not require to enforce any updates to the key derivation mechanism in wallets or any forks in the blockchain. This sounds very surprising as it is a well-known fact that the security of EdDSA is based on the computationally intractability assumption of the *Elliptic Curve Discrete Logarithm Problem (ECDLP)*, which can be broken using Shor's algorithm [(Shor, 1994)](https://doi.org/10.1109/SFCS.1994.365700) on quantum computers. It is worth emphasizing that the main concern in this document, as well as in Mysten Labs’ proposal, is the issue of *“sleeping accounts”* [(Baldimtsi et al., 2025)](https://eprint.iacr.org/2025/1368): accounts whose owners are unlikely to migrate their funds to post-quantum (PQ) accounts before Q-day.


## Preliminaries
Before presenting the idea, let me briefly review some preliminaries, including a comparison of the standard approaches to key derivation in ECDSA and EdDSA.

### ECDSA Key Pair Generation
In the ECDSA key generation, the secret key ($sk$) is randomly sampled from the interval $[1, q – 1]$, where $q$ is a sufficiently large prime number. Then, the public key $pk$ is derived directly as
$$
pk = sk\cdot B,
$$ where $B$ is the base point. The parameters $q$ and $B$ are specified according to the  parameters of the elliptic curve used. For example, [XRPL uses the famous](https://xrpl.org/docs/concepts/accounts/cryptographic-keys#secp256k1-key-derivation) **secp256k1** parameters specified by Certicom Corp. in [(Brown, 2010)](https://www.secg.org/sec2-v2.pdf). These parameters became more popular after they were [used in Bitcoin](https://en.bitcoin.it/wiki/Secp256k1). 


### EdDSA Key Pair Generation
The Mysten Labs' idea is based on EdDSA's standard for key generation, and it is different to ECDSA. [RFC 8032](https://datatracker.ietf.org/doc/html/rfc8032) ([Section 5.1.5](https://datatracker.ietf.org/doc/html/rfc8032#section-5.1.5)) describes the key generation of **Ed25519**, a famous parameter set of the EdDSA algorithm, which [XRPL aslo supports it](https://xrpl.org/docs/concepts/accounts/cryptographic-keys#ed25519-key-derivation). According to this standard, the secret key ($sk$) is a 32-byte (256 bits) randomly sampled number, and the public key ($pk$) is derived as 
$$
\begin{aligned}
ss &= \text{HashToScalar}(\textsf{SHA512}(sk)[0:32])\\
pk &= ss\cdot B,
\end{aligned}
$$ where $[0:32]$ denotes the lower 32 bytes of the $\textsf{SHA512}$'s output. This output truncation of $\textsf{SHA512}$ is referred to as "`SHA-512Half`" in [XRPL's documents](https://xrpl.org/docs/references/protocol/data-types/basic-data-types#hashes). Moreover, the $\text{HashtoScalar}$ procedure prunes (clamps) the 32-bytes result to derive a *secret scalar* $ss$ which is suitable for Ed25519 (The procedure and the rationale behind this is [described by Neil Madden](https://neilmadden.blog/2020/05/28/whats-the-curve25519-clamping-all-about/)). Finally, the derived $ss$ is multiplied by $B$, where $B$ denotes the base point of Ed25519.

### Elliptic Curve Discrete Logarithm Problem (ECDLP)
Roughly speaking, the ECDLP is defined as finding the scalar $s$ given $B$, denoting the base point, and a point $P$ in an elliptic curve, such that
$$
P = s \cdot B.
$$ECDLP is *apparently* intractable on classical computers if the elliptic curve parameters are carefully chosen. This problem is the basis for the security of ECDSA and EdDSA: given a public key $pk$ in either scheme, it is computationally infeasible to recover the corresponding private key $sk$ in ECDSA or secret scalar $ss$ in EdDSA.

However, Peter W. Shor first presented an algorithm in 1994 [(Shor, 1994)](https://doi.org/10.1109/SFCS.1994.365700) that can solve the ECDLP in polynomial time on quantum computers.

### Zero-knowledge Succinct Non-interactive Argument of Knowledge (zkSNARK)
Mysten Labs' method uses the idea of zkSNARKs. Here, I’ll briefly and informally explain what they are. A zkSNARK is a family of cryptographic protocols involving a prover and a verifier. The prover can generate a succinct, non-interactive argument showing they know a witness to an $\mathcal{NP}$ problem, without revealing the witness itself. The verifier, on the other hand, can check the validity of this argument without learning anything about the underlying witness.

Let me illustrate this with a simple example: suppose the prover knows the pre-image of a hash. That is, they know some value $w$ (witness) such that $$h = \mathcal{H}(w).$$ Using a zkSNARK protocol, the prover constructs a proof $\pi$ attesting to their knowledge of $w$, and sends $\pi$ along with $h$ (public input) to the verifier. The verifier then runs the verification algorithm and becomes convinced that the prover does know the pre-image of $h$.

The security of zkSNARKs against a malicious prover relies on a cryptographic hard problem, such as the aforementioned ECDLP, collision-resistance hash function (CRH), square span program (SSP), etc. Based on the underlying assumption, zkSNARKs can be categorized as either post-quantum secure or quantum-vulnerable.



# Post-Quantum Readiness

## Post-Quantum Readiness of EdDSA (Mysten Labs Idea)
While Shor's algorithm can solve ECDLP on quantum computers, it is assumed that many hash functions, such as $\textsf{SHA512}$, remain secure on Q-day. Consequently, even if a quantum adversary derives $ss$ from a EdDSA public key $pk$, it is still unable to recover $sk$. This let the real owner of the wallet be able to provide an **argument-of-ownership** (i.e., proof-of-ownership) of their keys, using zkSNARK protocols, even whether $ss$ is exposed or not. In other words, even if quantum adversaries break ECDLP and recover $ss$ from the public key $pk$, they still cannot determine the original secret key $sk$, which is the pre-image of $ss$. This allows the legitimate owner of an EdDSA-based account to generate a zero-knowledge argument of this knowledge, thereby proving possession of the account. This argument must be generated through **post-quantum secure zkSNARKs**, e.g., [STARK](https://eprint.iacr.org/2018/046), [Aurora](https://eprint.iacr.org/2018/828), [Ligero](https://eprint.iacr.org/2022/1608), etc.

There are multiple possible approaches to implementing this using existing zkSNARKs. Fortunately, since no changes to the current EdDSA are required, we likely have sufficient time to determine how to construct and verify this argument-of-ownership by Q-day. This argument can be used to transfer the funds from EdDSA accounts. Mysten Labs suggested an NP relation of $pk$ (public input) and $sk$ (private input or witness). However, by assuming that by Q-day $ss$ can be recovered from $pk$, a simpler relation, witch can be instantiated by proof of pre-image of $\textsf{SHA512}$ more efficiently.

## Post-Quantum Readiness of ECDSA
As we earlier discussed, $pk$ is directly derived from $sk$ in ECDSA. On Q-day, the quantum adversaries can derive the $sk$ from $pk$ and they will have the same knowledge as the real account owners (despite the EdDSA where the quantum adversary is able to recover $ss$ not $sk$). In this case, it is impossible for validators to distinguish the real ECDSA based account owners from quantum adversaries. Despite this problem existing in many blockchains, several communities recognized it long ago. 

### Bitcoin
The Bitcoin community recommended [avoiding key reuse](https://bitcoin.stackexchange.com/questions/118919/address-reuse-and-ecdsa-concern) and using unique addresses for receiving payments. In P2PKH and P2SH addresses, the ECDSA public key remains hidden (hashed) until the first time the funds are spent. Consequently, if the public key $pk$ has not yet been revealed, a quantum computer need to compute the pre-image of 160-bit hash 
$$\textsf{RIPEMD160}(\textsf{SHA256}(pk)),$$ to obtain $pk$ and, from it, derive the corresponding private key $sk$ using Shor's algorithm. Computing the pre-image ($pk$) would require approximately $2^{80}$ evaluations, using Grover’s algorithm ([Grover,1996](https://dl.acm.org/doi/10.1145/237814.237866)),  from P2PKH or P2SH addresses, i.e, the provided security level is 80 bits. While [some have raised concerns](https://x.com/myaksetig/status/1920918103883501827) about this relatively low security level under current cryptanalysis techniques, [NIST notes](https://csrc.nist.gov/csrc/media/Events/2024/fifth-pqc-standardization-conference/documents/papers/on-practical-cost-of-grover.pdf) that even for AES-128, which has a theoretical Grover cost of $2^{64}$, the practical impact is limited due to Grover’s poor parallelization properties. 

Despite this, it is worth recalling that Bitcoin addresses with revealed public keys remain vulnerable to quantum attacks. For example, [8.7% of all bitcoins](https://research.mempool.space/utxo-set-report/) are stored in the early Bitcoin address type, P2PK, where public keys are directly visible. Additionally, funds are also held in other address types where the public key has been exposed due to address reuse. [Deloitte reports that](https://www.deloitte.com/nl/en/services/consulting-risk/perspectives/quantum-computers-and-the-bitcoin-blockchain.html) 25% of all bitcoins are stored in public-key exposed addresses.

### Ethereum

Ethereum is almost similar, as it also uses ECDSA implemented in [`libsecp256k1`](https://github.com/bitcoin-core/secp256k1)  maintained by Bitcoin. However, Ethereum has one advantage and one disadvantage compared to Bitcoin. The advantage of Ethereum is that it computes $\textsf{Keccak256}$ of the public key to generate address, which provides a higher security level against Grover algorithm for the [scenario that the $pk$ is not revealed](https://ethresear.ch/t/how-to-hard-fork-to-save-most-users-funds-in-a-quantum-emergency/18901) through sending a transaction. The disadvantage is that Ethereum is an **account model** blockchain, where accounts prefer to reuse the same address. [Deloitte reports that](https://www.deloitte.com/nl/en/services/consulting-risk/perspectives/quantum-risk-to-the-ethereum-blockchain.html) 65% of all ether are stored in public-key exposed addresses.


### HD and Mnemonic Wallets
The Hierarchical deterministic (HD) wallet, explained in [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki), employs a system for deriving a tree of keypairs from a single seed to build a wallet structure on top of such a tree. The derivation of the master key and child keys in HD wallet is done through $\textsf{HMAC-SHA512}$. The presence of hash function in the key derivation lets us again use the argument-of-knowledge of the pre-image of a hash on Q-day, the knowledge quantum computers cannot achieve by breaking quantum-vulnerable cryptographic hard problems.

Additionally, in mnemonic wallets, explained in [BIP39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki), a secret binary seed is generated from a mnemonic sentence given to [PBKDF2](https://datatracker.ietf.org/doc/html/rfc2898#section-5.2), where the pseudorandom function is $\textsf{HMAC-SHA512}$ and the iteration count is 2048. Here, again the presence of hash function helps ECDSA based accounts to be able to generate argument-of-knowledge of the pre-image on Q-day. 

Vitalik, the founder of Ethereum, explains [here](https://ethresear.ch/t/how-to-hard-fork-to-save-most-users-funds-in-a-quantum-emergency/18901) how the BIP32 method of key generation can be employed to recover vulnerable accounts from a quantum emergency.

Furthermore, [EIP-7693](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki) presented a technique to efficiently generate the argument-of-knowledge of the secret pre-image in BIP32 or the mnemonic sentence in BIP39 on Q-day.

Both Bitcoin and Ethereum commonly rely on BIP32 together with BIP39. However, from the address alone, it is impossible to determine whether the underlying key was derived through BIP32/BIP39 or generated directly. Consequently, one cannot identify what fraction of bitcoins or ether is held in addresses originating from such derivation schemes.

## Post-Quantum Readiness of XRP Ledger
After this brief background, I can explain the quantum readiness of accounts on the XRP Ledger more clearly. I only need to investigate whether hash functions are used in the key derivation process or not. In this section, I will refer to specific lines of code or documentation whenever possible.


XRPL supports both **ECDSA** and **EdDSA**.  
- **ECDSA** is implemented over the `secp256k1` curve using [`libsecp256k1`](https://github.com/bitcoin-core/secp256k1), and  
- **EdDSA** is implemented over the `ed25519` curve using [`ed25519-donna`](https://github.com/floodyberry/ed25519-donna).  

As the Mysten Labs suggested (explained earlier), the EdDSA is quantum ready due to its standard key derivation mechanism. Some XRPL client libraries such as [`xrpl-py`](https://xrpl-py.readthedocs.io/en/stable/source/xrpl.wallet.html#xrpl.wallet.Wallet.create) and [`xrpl.js`](https://github.com/XRPLF/xrpl.js/blob/de28f4057e9ba3faa478957955d428dd89d2a5da/packages/xrpl/src/Wallet/index.ts#L36) use EdDSA by default. However, [**`rippled`**](https://github.com/XRPLF/rippled) uses ECDSA *by default* for [creating a new wallet](https://github.com/XRPLF/rippled/blob/b04d239926f265f2c7b5e1fd4a86b76d2479f322/src/xrpld/rpc/handlers/WalletPropose.cpp#L134) through RPC call (EdDSA can be still chosen). Also, `rippled` uses ECDSA to generate keypairs for [nodes](https://github.com/XRPLF/rippled/blob/b04d239926f265f2c7b5e1fd4a86b76d2479f322/src/xrpld/app/main/NodeIdentity.cpp#L54) and [validators](https://github.com/XRPLF/rippled/blob/b04d239926f265f2c7b5e1fd4a86b76d2479f322/src/xrpld/app/misc/detail/ValidatorKeys.cpp#L47). It worth noting that nodes and validators use their keys actively for [peer-to-peer authentication](https://xrpl.org/docs/concepts/networks-and-servers/peer-protocol#node-key-pair) and [validation messages](https://xrpl.org/docs/concepts/consensus-protocol/consensus-structure#compare-results), respectively; and not for keeping or transferring funds on the ledger. Therefore, We can assume that their algorithms will be changed to PQC algorithms when the quantum threats become more realistic. 

Alternatively, it is worth recalling that the main concern lies with *sleeping accounts*, those that keep funds on the ledger and are *unlikely* to migrate them to PQ accounts before Q-day. Determining the ratio of accounts on the XRP Ledger that use ECDSA versus EdDSA would be a valuable direction for future analysis.


As mentioned before, the post-quantum readiness of ECDSA accounts on XRPL depends on whether their key generation mechanism include hash functions, as in BIP39 and BIP39. **The good news** is that XRPL provides functionalities to generate new secret keys deterministically from a seed, which have been there since the beginning of XRPL. Even its genesis account, appeared in the first ledger, is generated from the seed value [`masterpassphrase`](https://github.com/XRPLF/rippled/blob/b04d239926f265f2c7b5e1fd4a86b76d2479f322/src/xrpld/app/ledger/Ledger.cpp#L184) from very beginning through calling [`generateKeyPair(KeyType type, Seed const& seed)`](https://github.com/XRPLF/rippled/blob/b04d239926f265f2c7b5e1fd4a86b76d2479f322/src/libxrpl/protocol/SecretKey.cpp#L369) method. 

Through the `generateKeyPair` method, if the chosen algorithm is ECDSA, the function [`deriveDeterministicRootKey(Seed const& seed)`](https://github.com/XRPLF/rippled/blob/afc05659edbc22d04b13402f157832778556f7b7/src/libxrpl/protocol/SecretKey.cpp#L86) computes the `SHA-512Half` of the given `seed` to derive the ECDSA root secret key. The `generateKeyPair` method is used as **the standard method** for generating a new (deterministic) wallet in [`walletPropose(Json::Value const& params)`](https://github.com/XRPLF/rippled/blob/b04d239926f265f2c7b5e1fd4a86b76d2479f322/src/xrpld/rpc/handlers/WalletPropose.cpp), which is called from the [`"wallet_propose"`](https://github.com/XRPLF/rippled/blob/b04d239926f265f2c7b5e1fd4a86b76d2479f322/src/xrpld/rpc/detail/Handler.cpp#L192) RPC call. The secret `seed` is either provided as a parameter to this RPC call or generated by `rippled` and returned. 

Whenever a client wants to sign a transaction, the seed is required (rather than, for instance, directly supplying an ECDSA secret key). From this seed, both the secret and public keys are derived. More specifically, in  `rippled`, the [`keypairForSignature`](https://github.com/XRPLF/rippled/blob/b04d239926f265f2c7b5e1fd4a86b76d2479f322/src/xrpld/rpc/detail/RPCHelpers.cpp#L795) function gets `seed` through an [RPC sign request](https://xrpl.org/docs/references/http-websocket-apis/admin-api-methods/signing-methods/sign) and re-generates keypairs through passing the `seed` to [`generateKeyPair`](https://github.com/XRPLF/rippled/blob/b04d239926f265f2c7b5e1fd4a86b76d2479f322/src/xrpld/rpc/detail/RPCHelpers.cpp#L927C5-L927C11). 


In summary, XRPL's support for both EdDSA and ECDSA allows EdDSA-based accounts to be considered post-quantum ready, as their standard employs $\textsf{SHA512}$ for secret key derivation. Moreover, XRPL's native standard for deriving ECDSA keys from a `seed`, from very beginning, also using the same hash function, **provides post-quantum readiness for all ECDSA-based accounts as well**. I also reviewed the `rippled` codebase to confirm that users have to comply with the XRPL standard. They are required to provide the `seed` when signing transactions, rather than supplying only the ECDSA secret key. Accordingly, it is reasonable to assume that all XRPL account owners preserve the `seed` corresponding to their account. Therefore, on Q-day, they would be able to provide a post-quantum secure argument-of-knowledge to prove real ownership of their account, as quantum computers may not be able to recover the `seed` due to the post-quantum resistance of hash functions (e.g., $\textsf{SHA512}$).

![Gemini_Generated_Image_gritzzgritzzgrit](https://hackmd.io/_uploads/H135HwXFee.jpg)



# Conclusion

In this post, I examined the quantum-readiness of the XRPL ledger based on its native account creation standard, which relies on **hash functions**. This property provides reassurance for Q-day, as real account owners will still be able to prove their ownership through a quantum-resistant argument-of-knowledge.  

Building on this readiness, the next step is to design a secure and efficient mechanism for generating such arguments of knowledge. With current technologies, post-quantum secure proof systems such as STARKs or Aurora zkSNARKs may serve as practical candidates for this purpose.

