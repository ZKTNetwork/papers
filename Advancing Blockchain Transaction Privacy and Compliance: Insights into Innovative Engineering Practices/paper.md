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

TODO

### 4.3.1 Performance Comparision

TODO

## 5. ASP Part @Xxiang

association set providers(ASPs).

## 6. Client-Side Acceleration Solution


## References

1. plookup: A simplified polynomial protocol for lookup tables https://eprint.iacr.org/2020/315.pdf
