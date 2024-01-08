# Advancing Blockchain Transaction Privacy and Compliance: Insights into Innovative Engineering Practices

## 1. Introduction

Blockchain privacy and regulatory compliance, a dynamic shift is underway, inspired by the pivotal works of Vitalik Buterin and [Ameen Soleimani](https://ethresear.ch/u/ameensol). Their paper, "[Blockchain Privacy and Regulatory Compliance: Towards a Practical Equilibrium](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4563364)," alongside the insightful [forum post](https://ethresear.ch/t/permissioned-privacy-pools/13572) on permissioned privacy pools, paves the way for a nuanced understanding of the delicate balance between maintaining transactional privacy and adhering to regulatory norms. These resources offer a profound exploration of the challenges and potential solutions in harmonizing privacy with compliance in the evolving blockchain landscape.

## 2. System Architecture

At the forefront of this evolution is Vala, an innovative settlement layer that embodies the principles of compliant privacy transactions. Vala integrates a multi-chain cross-chain module, fostering seamless interoperability across various blockchain networks. This feature is pivotal in enabling a boundless crypto ecosystem, where asset transfer and management are not confined to the boundaries of a single blockchain.

Central to Vala's architecture is the Privacy Pool, a sophisticated mechanism that allows users to engage in UTXO (Unspent Transaction Output) model transactions. This model is instrumental in enhancing user privacy while ensuring transactions remain transparent and compliant with regulatory standards. The UTXO model, a hallmark of blockchain technology, offers an efficient way to track asset ownership and transfer, laying the groundwork for secure and private transactions within the Vala ecosystem.

![Tree](./tree.jpeg)

## 3. Technology Stack

The technology stack for zkUSD Vala is anchored by zkSNARK for privacy preservation. The UTXO Proof uses a Merkle Tree structure, while Membership Proof employs Plonk + Plookup[^1] for enhanced search speed. The project's zero-knowledge proofs part is implemented in Rust, optimizing performance and security. For the frontend, proof generation utilizes WebAssembly compiled from our Rust code, significantly accelerating the proof generation process for users. This stack represents a blend of advanced cryptographic techniques and efficient programming solutions, tailored for robust and fast privacy transactions.


## 4. Component Details

### 4.1 Infrastructure

The users' fungible assets are hashed into a Binary Merkle Tree. The leaf node is initialized with a constant value $H'=\mathrm{Hash}(0)$ until any asset occupies the slot of the leaf, then the leaf node should be updated to $H=\mathrm{Hash}(identifier \ | \ amount \ | \ commitment)$, where $identifier$ is an associated tag of this asset (like the user's address), $amount$ is the quantity of the asset stored at the leaf node, and $commitment = \mathrm{Hash}(secret)$, with $secret$ being the key of the user who owns the asset.

For an unused leaf node slot, the user can deposit an asset into the pool and take the slot while providing the asset and commitment. For withdrawal, the user should provide the withdrawal amount, new leaf hash, and the snark proof.

![merkletree](./merkletree.svg)

### 4.2 Proof of ULO (Unspent Leaf Output)

The design of the withdrawal mechanism is influenced by the Bitcoin UTXO (Unspent Transaction Output) model, which we've adapted into what we call ULO (Unspent Leaf Output). Users select a set of leaf nodes that indicate their assets to use as inputs, then hash the remaining balance into a new leaf node as output. The difference between the total input and the output is the amount of the withdrawal.

Now let the ULO source list be $L$, for each ULO with an index $i\in L$, we must first verify its existence and then calculate the corresponding leaf's nullifier within the circuit as follows:

$$
\begin{aligned}
commitment_i&=\mathrm{Hash}(secret_i)
\\
nullifier_i&=\mathrm{Hash}(secret_i^{-1})
\\
leaf_i&=\mathrm{Hash}(identifier_i \ | \ amount_i \ | \ commitment_i)
\\
root&=\mathrm{MerklePath}(leaf_i, \ elements)
\end{aligned}
$$

Next, we need to demonstrate that the total input amount is equal to the total output amount within the circuit, and then generate the leaf output. The following equations represent this:

$$
\begin{aligned}
\sum_{i\in L}amount_i&=amount_w + amount_o
\\
commitment&=\mathrm{Hash}(secret)
\\
leaf&=\mathrm{Hash}(identifier \ | \ amount_o \ | \ commitment)
\end{aligned}
$$

Where $amount_w$ is the withdrawal amount, $amount_o$ is the output amount.

**Note**
In the above process, we set $\{nullifier_i\}_{i\in L}$, $leaf$, $root$ and $amount_w$ as public variables from the circuits.

### 4.3 Proof of membership

#### 4.3.1 Why not Merkle proof

Proof of membership in Vala lies in demonstrating that the inputs come from an arbitrary set constructed by the user, meanwhile ensuring that this set is publicly available to anyone. Typically, the set can be organized into a new Merkle Tree and the user should prove that each input is also a leaf node of the Merkle Tree (Indeed, privacy-focused solutions like Tornado Cash v2, and Privacy Pools have adopted this kind of design).

Apparently, if the set is too small, the user's privacy is compromised. If the set is too large, generating the corresponding Merkle tree incurs a huge gas cost on EVM, since ZK-friendly hash functions (Poseidon, Rescue) are not integrated by EVM primitively while creating Merkle tree has a complexity of O(n) hash.

We hereby adopt the Plookup to construct proof of membership, instead of Merkle Tree. Let’s take a look at the design idea of Plookup:

*"we precompute a lookup table of the legitimate (input, output) combinations; and the prover argues the witness values exist in this table."*

There are **2** main advantages of Plookup in our membership proof.
- In the proving phase, we do not need to perform existence proofs for each ULO, which significantly reduces the circuit size.
- In the verification phase, we do not need to perform hash operations in EVM. Only one elliptic curve pairing is required.

#### 4.3.2 Plookup implementation

Assuming the user provides an identifier list of size $m$ from the source ULOs, denoted as $t$, then we pad the set $t$ with 0 until it satisfies the size of circuit size $n$.

$$
t=\{id_0,id_1,...,0,...,0\}
$$

Let $f$ be the query table:

$$
f_i =
\left\{\begin{matrix}
c_i, &\ \mathrm{if \ the} \ i\mathrm{\ gate \ is \ a \ lookup \ gate}
\\
0, &\ \mathrm{otherwise}
\end{matrix}\right.
$$

Where $c_i$ is the witness variable defined in the arithmetic gate of Plonk. When $c_i$ represents the identifier variable, $f_i$ equals to $c_i$.

Permutation polynomial is defined as

$$
\begin{aligned}
z_2(X)=&(d_2X^2+d_1X+d_0)Z_H(X)+L_1(X)
\\
&+\sum_{i=1}^{n-1}\left(L_{i+1}(X)\prod_{j=1}^{i}\frac{(1+\delta)(\varepsilon+f_j)(\varepsilon(1+\delta)+t_j+\delta t_{j+1})}{(\varepsilon(1+\delta)+s_{2j-1}+\delta s_{2j})(\varepsilon(1+\delta)+s_{2j}+\delta s_{2j+1})}\right)
\end{aligned}
$$

Extend the quotient polynomial of Plonk, which is similar to the Plonkup zk-snark[^2]:

$$
q(X)=\frac{1}{Z_H(X)}
\left(
\begin{aligned}
&a(X)b(X)q_M(X)+q(X)q_L(X)+b(X)q_R(X)+c(X)q_O(X)+q_C(X)+\mathrm{PI}(X)
\\
&+ \ ... \ ...
\\
&+q_K(X)(c(X)-f(X))\alpha^3
\\
&+z_2(X)(1+\delta)(\varepsilon+f(X))(\varepsilon(1+\delta)+t(X)+\delta t(\omega X))\alpha^4
\\
&-z_2(\omega X)(\varepsilon(1+\delta)+h_1(X)+\delta h_2(X))(\varepsilon(1+\delta)+h_2(X)+\delta h_1(\omega X))\alpha^4
\\
&+(z_2(X)-1)L_1(X)\alpha^5
\\
&+q_T(X)t(X)\alpha^6
\end{aligned}
\right)
$$

Where the selector $q_K(X)$ switches on/off the lookup gate, $q_T(X)$ controls the padding elements should be all 0.

During the verification phase, in addition to the zk-SNARK proof verification, we just need to verify the opening proof of $t(X)$ at $\{\omega, \omega^2, ..., \omega^m\}$, without any hash operations on EVM.

**Note**: 
We need to construct an aggregated opening proof, which is called a multi-point opening, and use only one elliptic curve pairing operation.

### 4.3.3 Performance Comparision

TODO

## 5. ASP Part @Xxiang

association set providers(ASPs).

## 6. Client-Side Acceleration Solution


## References

[^1]:[plookup: A simplified polynomial protocol for lookup tables](https://eprint.iacr.org/2020/315.pdf)
[^2]:[plonkup: PlonKup: Reconciling PlonK with plookup](https://eprint.iacr.org/2022/086)
