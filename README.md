# Explorations Towards Zero-Knowledge Proof Verification on Constrained Devices

This research[^1] explores methods for efficiently verifying zero-knowledge (ZK) proofs on resource-constrained devices, such as secure elements. Our focus is on the **Groth16**[^2] proof system over the **BN254** curve, a widely adopted standard in ZK systems like **SNARKs** and **STARKs**, largely due to its near-optimality and support from Ethereum's EIP-197 precompiles. We target a scenario where a powerful but potentially malicious host device delivers proofs to a secure element for verification.

## Problem Statement

Groth16 requires the verifier to store (or, perhaps, receive and authenticate) one FP2 field element for each public input supported by a circuit. Each of these elements must then be multiplied by its corresponding public input. Without loss of generality we ignore this overhead, since a straightforward recursive verification circuit can be constructed which takes in only a single public field element containing a commitment to a specific verification key and public input. By recursively proving knowledge of a correct, arbitrary Groth16 proof, such a circuit reduces the memory and processing burdens upon the constrained verifier, by avoiding both extra FP2 multiplications and storage or loading of multiple verification keys.

The remaining operations which must be checked by the verifier to ensure validity of a proof consist of one scalar multiplication of an FP2 element, two FP12 additions, and three pairing computations. The operations other than the pairings themselves are trivial and straightforward to directly verify even on a highly constrained device. The pairings, unfortunately, are not.

A standard optimal ate pairing consists of two steps, the Miller loop and the final exponentiation, of roughly equivalent computational burden. The final exponentiation uses a 2790-bit exponent, and requires 1376 FP12 multiplications and 2790 FP12 squarings to compute using the standard exponentiation-by-squaring algorithm. Each of these operations decomposes to 144 multiplications over the base prime field in a naive implementation, and the equivalent of approximately 41 multiplications in an optimized one. This puts the total burden for just the final exponentiation of a pairing computation at a 175,230 254-bit multiplications, with the total 3-pairing computation approximately equivalent to a million multiplication operations even after optimization.

## Initial Benchmarking: Direct Computation is Infeasible

To gauge feasibility, we implemented the final exponentiation step on an **NXP J3R180 JCOP4** card, using JCMathLib[^3] to leverage the card's RSA cryptographic accelerator to perform the base prime-field arithmetic.

The full computation too long to measure effectively. We shortened the test to a single FP12 multiplication and a single FP12 squaring, which took approximately **34 seconds**. This puts each base field multiplication at approximately ~120ms on this platform. Even with a fully optimized FP12 multiplication algorithm (which would be about 70% faster), our benchmark results suggest that the card would still take almost **two days** to complete three pairings.

## Pairing Delegation

### DCKKS20

This result indicated strongly that an approach other than direct computation was required. DCKKS20[^4] presented an algorithm for the delegation of the computation of a pairing with public inputs, which fits our use case. This algorithm enables the verifier to check that the pairing was computed correctly by a potentially-malicious host using only a single 254-bit exponentiation and without running the Miller loop at all. Their approach requires a 384-byte precomputed secret pairing result, but this can be practically delivered in encrypted, authenticated form by the untrusted host. For reference, a typical payment card has a transaction limit of 2^16 transactions, which is clearly seen as acceptably high; the precomputation for this many transactions would occupy only 24MiB, and the host device would only have to fetch a little over a kilobyte of fresh material from remote storage for each proof.

To test the feasibility of this approach, we wrote another simple proof-of-concept benchmark for the **Ledger Flex**, a device built on the ST33K1M5C secure MCU. This platform is quite a bit faster than our original target. The benchmark used the Ledger SDK's cryptographic API to perform optimized modular multiplication and squaring, and simulated the performance characteristics of an optimal FP12 exponentiation loop. Each pairing check would require two-plus-the-exponent's-bit-count FP12 multiplications (each of which costs 39 multiplications and 12 squarings in FP), as well as 256 FP12 squarings (23 multiplications and 12 squarings in FP). Roughly half the bits can be expected to be set in the exponent in the average case, yielding a final cost of 10958 multiplications and 4632 squarings in FP to verify a single pairing. This operation took **24 seconds** on the test device. Using the rule-of-thumb that FP squarings are worth 80% of an FP multiplication, this puts the cost of a single multiplication operation at ~1.6ms. This represents a total potential speedup of ~5500x: ~70x from the improved hardware, ~3x from the use of optimized FP12 tower multiplication, ~20x from the use of the DCKKS20 delegation algorithm, and an additional ~1.3x from changing our evaluation target from worst-case to average-case characteristics. Unfortunately, even 72 seconds is too long of a verification time to be practically useful in a user-facing application.

### AmorE

A further advancement in delegated pairings over DCKKS20 has recently been made in the form of the "AmorE" algorithm discussed in KAPRH25[^5]. It offers batched pairing verification, enabling a single round of the protocol to verify all three required pairings at reduced cost, and makes efficiency optimizations by relaxing its security target slightly. Rather than providing full 128-bit statistical security against an unbounded adversary, it instead makes the practical assumptions that the verification protocol will incorporate a timeout and that the adversary will have less power than the current entire hash rate of the Bitcoin network. (The Ledger's hardware watchdog will reset the device if its calculations take longer than about 30 seconds, conveniently ensuring the first point.)

Under these assumptions, AmorE reduces the verification requirements for the entire set of three pairings to a few relatively easy FP2 and FP6 multiplications and 271 FP12 multiplications. This represents a further **~3x speedup** over DCKKS20 in the worst case.

We attempted to update our benchmark application to with the AmorE authors' own implementation of their protocol in the RELIC[^6] cryptographic toolkit. Unfortunately, cross-compilation issues prevented us from collecting real-world data by the project deadline. Theoretical estimates suggest that a full three-pairing verification would complete in approximately **32 seconds**.

## Future Work

### Software-accelerated Math?

It is nevertheless (though counterintuitively) possible that the RELIC library's use of ARM assembly for the base field multiplication rather than the usual cryptographic hardware accelerator could provide a significant speedup. These primitives are primarily designed for multiplication over 1024-bit up to 4096-bit prime fields, because they were originally intended to support the large moduli used with RSA. It's quite plausible that their performance characteristics could be worse than plain CPU integer operations when used with the smaller 254-bit modulus. (BN254 field elements span only 8 32-bit words, and the Ledger's Cortex-M35P platform boasts single-cycle 32-bit multiplication.)

### Tiny STARKs

It is possible that an extremely-specialized STARK to offload computation of the remaining FP12 modular exponentiations to the host. Our applications don't benefit from zero-knowledge or non-interactivity, simplifying both proving and verification, and the hash operations required to verify Merkle commitments to FRI polynomials are very effectively accelerated on secure element hardware. Hash throughput is often significantly faster than large-integer modular arithmetic, and STARK verification cost would be entirely dominated by the FP multiplications required to verify correct mixing of the constraint polynomial by the host. This cost is linear in the number of constraints.

STARKs typically use large numbers of constraints, but the extremely simple  iterative process of exponentiation-by-squaring should be possible in a STARK with very few constraints. We considered this approach earlier in our research, as a possible approach for delegating final exponentiation on the smart card platform, but it would have taken impractically many registers and constraints to track the bit-shifting of the applicable 2790-bit exponent. The delegated pairing algorithms we explored later, however, only require exponents which fit in a single field element, which would greatly simplify the construction of such a STARK.

This approach would only be practical if kept to a such a low number of constraints the client device had to process that verification of the mixed constraint polynomial was significantly cheaper than the number of multiplications required for direct computation of an exponentiation. If that threshold is achieved, however, FRI batching would conveniently allow the polynomials for every exponentiations needed for all three pairings to be checked during a single interactive phase.

---

[^1]: Some initial work with JCMathLib was done leading up to and during ZKHack Berlin. This was limited to adding support for the fundamental mathematical primitives such as pairing-friendly curves required for the present work and discovering that the low cofactor of BN254 made it a better with for the JCOP APIs than BLS12-381. This led in part to the decision to target the less-secure field for these experiements.

[^2]: Groth, J. (2016). On the size of pairing-based non-interactive arguments (Cryptology ePrint Archive, Paper 2016/260). https://eprint.iacr.org/2016/260

[^3]: Mavroudis, V., & Svenda, P. ([2020](https://doi.org/10.48550/arXiv.1810.01662)). [JCMathLib](https://github.com/OpenCryptoProject/JCMathLib): wrapper cryptographic library for transparent and certifiable JavaCard applets. In 2020 IEEE European Symposium on Security and Privacy Workshops (EuroS&PW) (pp. 89-96). IEEE.

[^4]: Di Crescenzo, G., Khodjaeva, M., Kahrobaei, D., & Shpilrain, V. ([2020](https://doi.org/10.1007/978-3-030-68487-7_6)). [Secure and Efficient Delegation of Pairings with Online Inputs. In Smart Card Research and Advanced Applications](https://web.archive.org/web/20250706025116/https://cardis2020.its.uni-luebeck.de/files/CARDIS2020_DiCrescenzo_SecureAndEfficientDelegation_paper.pdf) (pp. 84-99). CARDIS 2020.

[^5]: Keilty, A. P., Aranha, D. F., Pagnin, E., & Rodríguez-Henríquez, F. (2025). That’s AmorE: Amortized Efficiency for Pairing Delegation (Cryptology ePrint Archive, Paper 2025/542). https://eprint.iacr.org/2025/542

[^6]: https://github.com/relic-toolkit/relic
