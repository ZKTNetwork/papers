# Advancing Blockchain Transaction Privacy and Compliance: Insights into Innovative Engineering Practices

## 1. Introduction

Blockchain privacy and regulatory compliance, a dynamic shift is underway, inspired by the pivotal works of Vitalik Buterin and [ameensol](https://ethresear.ch/u/ameensol). Their paper, "[Blockchain Privacy and Regulatory Compliance: Towards a Practical Equilibrium](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4563364)," alongside the insightful [forum post](https://ethresear.ch/t/permissioned-privacy-pools/13572) on permissioned privacy pools, paves the way for a nuanced understanding of the delicate balance between maintaining transactional privacy and adhering to regulatory norms. These resources offer a profound exploration of the challenges and potential solutions in harmonizing privacy with compliance in the evolving blockchain landscape.

## 2. System Architecture

At the forefront of this evolution is Vala, an innovative settlement layer that embodies the principles of compliant privacy transactions. Vala integrates a multi-chain cross-chain module, fostering seamless interoperability across various blockchain networks. This feature is pivotal in enabling a boundless crypto ecosystem, where asset transfer and management are not confined to the boundaries of a single blockchain.

Central to Vala's architecture is the Privacy Pool, a sophisticated mechanism that allows users to engage in UTXO (Unspent Transaction Output) model transactions. This model is instrumental in enhancing user privacy while ensuring transactions remain transparent and compliant with regulatory standards. The UTXO model, a hallmark of blockchain technology, offers an efficient way to track asset ownership and transfer, laying the groundwork for secure and private transactions within the Vala ecosystem.

![Tree](./tree.jpeg)

## 3. Technology Stack

The technology stack for zkUSD Vala is anchored by zkSNARK for privacy preservation. The UTXO Proof uses a Merkle Tree structure, while Membership Proof employs Plonk + Plookup[1] for enhanced search speed. The project's zero-knowledge proofs part are implemented in Rust, optimizing performance and security. For the frontend, proof generation utilizes WebAssembly  compiled from our Rust code, significantly accelerating the proof generation process for users. This stack represents a blend of advanced cryptographic techniques and efficient programming solutions, tailored for robust and fast privacy transactions.


## 4. Component Details

### 4.1 Infrastructure

The users' fungible assets are hashed into in a Binary Merkle Tree. The leaf node is initialized with a constant value until any asset occupies the slot of the leaf, then the leaf node should be updated to $H_i=\mathrm{Hash}(tag_i \ | \ amount)$, where $i$ is the leaf node index, $amount$ is the quantity of assets stored at that leaf node, and $tag_i = \mathrm{Hash}(i \ | \ secret)$, with $secret$ being the key of the user who owns the assets.

Let the latest available leaf index is $i$. The user can deposit an asset into the pool while providing the asset and the $tag_i$, then computes the leaf hash and updates the Merkle Tree. For withdrawal, the user provides the withawal amount, $tag_i$ and snark proof.

![merkletree](./merkletree.svg)

### 4.2 Proof of ULO (Unspent Leaf Output)

The design of the withdrawal mechanism is influenced by the Bitcoin UTXO (Unspent Transaction Output) model, which we've adapted into what we call the Unspent Leaf Output (ULO). Users select a set of indexes that represent their own assets to use as inputs. They then hash the remaining balance into a new available leaf node as output. The difference between the total input and the output is the amount the user withdraws.

For an Unspent Leaf Output (ULO) with index (i), we must first verify its existence and then calculate the corresponding leaf's nullifier within the circuit as follows:

$$
\begin{aligned}
tag_i&=\mathrm{Hash}(i \ | \ secret)
\\
nullifier_i&=\mathrm{Hash}(secret \ | \ i)
\\
leaf_i&=\mathrm{Hash}(tag_i \ | \ amount_i)
\\
root&=\mathrm{MerklePath}(leaf_i, \ elements)
\end{aligned}
$$

Next, we need to demonstrate that the total input amount is equal to the total output amount within the circuit. This is represented by the following equations:

$$
\begin{aligned}
\sum_{i\in\{\}}amount_i&=withdrawal + amount_k
\\
tag_k&=\mathrm{Hash}(k \ | \ secret)
\\
leaf_k&=\mathrm{Hash}(tag_k \ | \ amount_k)
\end{aligned}
$$

Note: in above process, we set $k$, $nullifier$, $leaf_k$, and $root$ as public variables of the circuits, where k is the lastest available leaf index.

### 4.3 Proof of membership

#### 4.3.1 Why not merkle proof

Proof of membership in zkUSD lies in demonstrating that an input comes from an arbitrary set constructed by user, meanwhile ensuring that this set is publicly available to anyone. Typically, the approach is to organize the set into a new Merkle Tree and prove that the input is also a leaf node of that Merkle Tree (Indeed, privacy-focused solutions like Tornado Cash v2 have adopted this kind of design).

However, if the set is too small, the user's privacy is compromised, else the set is too large, generating the corresponding merkle tree incurs a huge gas cost on blockchain, since ZK-friendly hash (Poseidon, Rescue) are not integrated by EVM primitively, while creating merkle tree has a hash complexity of O(n).

We hereby adopt the Plookup to construct proof of membership, instead of Merkle Tree. Let’s take a look at the design idea of Plookup:

*"we precompute a lookup table of the legitimate (input, output) combinations; and the prover argues the witness values exist in this table."*

#### 4.3.2 Plookup implementation

Assuming the user provides a number of source ULOs, denoted as $m$, and we denote this set as $t$, you would pad the set $t$ with zeros until it satisfies the number of elements in the subgroup. This padding ensures that the set size matches the required size for the plookup permutation.

$$
t=\{i,j,...,0,...,0\}
$$

$f$ is the query table:

$$
f_i =
\left\{\begin{matrix}
c_i, &\ \mathrm{if \ the} \ i\mathrm{\ gate \ is \ a \ lookup \ gate}
\\
t_n, &\ \mathrm{otherwise}
\end{matrix}\right.
$$

Permutation polynomial is defined as

$$
\begin{aligned}
z_2(X)=&(d_2X^2+d_1X+d_0)Z_H(X)+L_1(X)
\\
&+\sum_{i=1}^{n-1}\left(L_{i+1}(X)\prod_{j=1}^{i}\frac{(1+\delta)(\varepsilon+f_j)(\varepsilon(1+\delta)+t_j+\delta t_{j+1})}{(\varepsilon(1+\delta)+s_{2j-1}+\delta s_{2j})(\varepsilon(1+\delta)+s_{2j}+\delta s_{2j+1})}\right)
\end{aligned}
$$

Extend the quotient polynomial of Plonk, this is similar to the Plonkup zk-snark:

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
**Note**: Actually we need to construct an aggregated opening proof, which is called multi-point opening and use only one elliptic curve pairing operation.

### 4.3.3 Performance Comparision

TODO

## 5. ASP Part @Xxiang

association set providers(ASPs).

## 6. Client-Side Acceleration Solution


## References

1. plookup: A simplified polynomial protocol for lookup tables https://eprint.iacr.org/2020/315.pdf
