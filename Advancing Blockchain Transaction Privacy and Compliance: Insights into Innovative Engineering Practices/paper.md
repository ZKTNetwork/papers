# Advancing Blockchain Transaction Privacy and Compliance: Insights into Innovative Engineering Practices

## 1. Introduction

Blockchain privacy and regulatory compliance, a dynamic shift is underway, inspired by the pivotal works of Vitalik Buterin and [ameensol](https://ethresear.ch/u/ameensol). Their paper, "[Blockchain Privacy and Regulatory Compliance: Towards a Practical Equilibrium](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4563364)," alongside the insightful [forum post](https://ethresear.ch/t/permissioned-privacy-pools/13572) on permissioned privacy pools, paves the way for a nuanced understanding of the delicate balance between maintaining transactional privacy and adhering to regulatory norms. These resources offer a profound exploration of the challenges and potential solutions in harmonizing privacy with compliance in the evolving blockchain landscape.

## 2. System Architecture

At the forefront of this evolution is Vala, an innovative settlement layer that embodies the principles of compliant privacy transactions. Vala integrates a multi-chain cross-chain module, fostering seamless interoperability across various blockchain networks. This feature is pivotal in enabling a boundless crypto ecosystem, where asset transfer and management are not confined to the boundaries of a single blockchain.

Central to Vala's architecture is the Privacy Pool, a sophisticated mechanism that allows users to engage in UTXO (Unspent Transaction Output) model transactions. This model is instrumental in enhancing user privacy while ensuring transactions remain transparent and compliant with regulatory standards. The UTXO model, a hallmark of blockchain technology, offers an efficient way to track asset ownership and transfer, laying the groundwork for secure and private transactions within the Vala ecosystem.

![Tree](./tree.jpeg)

## 3. Technology Stack

The technology stack for zkUSD Vala is anchored by zkSNARK for privacy preservation. The UTXO Proof uses a Merkle Tree structure, while Membership Proof employs Plonk + Plookup[^1] for enhanced search speed. The project's zero-knowledge proofs part are implemented in Rust, optimizing performance and security. For the frontend, proof generation utilizes WebAssembly  compiled from our Rust code, significantly accelerating the proof generation process for users. This stack represents a blend of advanced cryptographic techniques and efficient programming solutions, tailored for robust and fast privacy transactions.


## 4. Component Details

### 4.1 Infrastructure

The users' fungible assets are hashed into in a Binary Merkle Tree. The leaf node is initialized with a constant value $H'=\mathrm{Hash}(0)$ until any asset occupies the slot of the leaf, then the leaf node should be updated to $H=\mathrm{Hash}(identifier \ | \ amount \ | \ commitment)$, where $identifier$ is an associated tag of this asset (like user's address), $amount$ is the quantity of the asset stored at the leaf node, and $commitment = \mathrm{Hash}(secret)$, with $secret$ being the key of the user who owns the asset.

For an unused leaf node slot, the user can deposit an asset into the pool and take the slot while providing the asset and commitment. For withdrawal, the user should provide the withdraw amount, new leaf hash and the snark proof.

![merkletree](./merkletree.svg)

### 4.2 Proof of ULO (Unspent Leaf Output)

The design of the withdrawal mechanism is influenced by the Bitcoin UTXO (Unspent Transaction Output) model, which we've adapted into what we call ULO (Unspent Leaf Output). Users select a set of leaf nodes that indicate their own assets to use as inputs, then hash the remaining balance into a new leaf node as output. The difference between the total input and the output is the amount of the withdrawal.

Now let the ULO source list be $L$, for each ULO with index $i\in L$, we must first verify its existence and then calculate the corresponding leaf's nullifier within the circuit as follows:

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

Next, we need to demonstrate that the total input amount is equal to the total output amount within the circuit, then generate the leaf output. This is represented by the following equations:

$$
\begin{aligned}
\sum_{i\in L}amount_i&=amount_w + amount_o
\\
commitment&=\mathrm{Hash}(secret)
\\
leaf&=\mathrm{Hash}(identifier \ | \ amount_o \ | \ commitment)
\end{aligned}
$$

Where $amount_w$ is the withdraw amount, $amount_o$ is the output amount.

**Note**
In above process, we set $\{nullifier_i\}_{i\in L}$, $leaf$, $root$ and $amount_w$ as public variables from the circuits.

### 4.3 Proof of membership

#### 4.3.1 Why not merkle proof

Proof of membership in Vala lies in demonstrating that the inputs come from an arbitrary set constructed by user, meanwhile ensuring that this set is publicly available to anyone. Typically, the set can be organized into a new Merkle Tree and user should prove that each input is also a leaf node of the Merkle Tree (Indeed, privacy-focused solutions like Tornado Cash v2, Privacy Pools have adopted this kind of design).

Apparently, if the set is too small, the user's privacy is compromised. If the set is too large, generating the corresponding merkle tree incurs a huge gas cost on EVM, since ZK-friendly hash functions (Poseidon, Rescue) are not integrated by EVM primitively, while creating merkle tree has a complexity of O(n) hash.

We hereby adopt the Plookup to construct proof of membership, instead of Merkle Tree. Let’s take a look at the design idea of Plookup:

*"we precompute a lookup table of the legitimate (input, output) combinations; and the prover argues the witness values exist in this table."*

There are **2** main advantages of Plookup in ower membership proof.
- In the proving phase, we do not need to perform existence proofs for each ULO, which significantly reduces the circuit size.
- In the verification phase, we do not need to perform hash operation in EVM. In fact, only one elliptic curve pairing is required.

#### 4.3.2 Plookup implementation

Assuming the user provides an identifier list of size $m$ from the source ULOs, denoted as $\mathbf{t}$, then we pad the set $\mathbf{t}$ with 0 until it satisfies the size of circuit size $n$.

$$
\mathbf{t}=\{\mathrm{id}_0,\mathrm{id}_1,...,0,...,0\}
$$

Let $\mathbf{f}$ be the query table:

$$
f_i = q_{Ki} \cdot c_i =
\left\{
\begin{matrix}
c_i, &\ \mathrm{if \ the} \ i\mathrm{\ gate \ is \ a \ lookup \ gate}
\\
0, &\ \mathrm{otherwise}
\end{matrix}
\right.
$$

Where $c_i$ is the output witness defined in arithmetic gate of Plonk, we activate it when $c_i$ represents the identifier witness. $q_{Ki} \in \{0,1\}$ is the selector that switches on/off the lookup gate.

Furthermore, we define $\mathbf{q_T}$ to satisfy $q_{Ti}\cdot t_i = 0$:

$$
q_{Ti}=
\left\{
\begin{matrix}
0, &\ i \leq m
\\
1, &\ i > m
\end{matrix}
\right.
$$

During the verification phase, besides the zk-SNARK proof verification, we  can verify the opening proof of $\mathbf{t}$ at $\{1, \omega, \omega^2, ..., \omega^{m-1}\}$ to confirm the correctness of each identifier in $\mathbf{t}$, without any hash operations.

**Note**: 
Actually we need to construct an aggregated opening proof, which is called multi-point opening and use only one elliptic curve pairing operation.

### 4.3.3 Performance Comparision

TODO

## 5. Protocol

We now describe a protocol that referred to Plonkup[^2] customized for zkUSD.

#### Common Referenced Input

$$
n,\ [1]_1,[x]_1,...,[x^{n+5}]_1,[1]_2,[x]_2,
\\
q_L(X)=\sum_{i=1}^{n}q_{Li}L_i(X),\ q_R(X)=\sum_{i=1}^{n}q_{Ri}L_i(X),\ q_O(X)=\sum_{i=1}^{n}q_{Li}L_i(X),
\\
q_C(X)=\sum_{i=1}^{n}q_{Ci}L_i(X),\ q_K(X)=\sum_{i=1}^{n}q_{Ki}L_i(X),\ q_T(X)=\sum_{i=m+1}^{n}L_i(X),
\\
S_{\sigma 1}(X)=\sum_{i=1}^{n}\sigma_{1i}L_i(X),\ S_{\sigma 2}(X)=\sum_{i=1}^{n}\sigma_{2i}L_i(X),\ S_{\sigma 3}(X)=\sum_{i=1}^{n}\sigma_{3i}L_i(X)
$$

#### Public input

$$
\mathrm{PI}(X)=\sum_{i=1}^{l}w_iL_i(X)
$$

### 5.2 Proving Process

#### Round 1

- Generate randome blinding scalars $b_1,...,b_6$.
- Compute the wire polynomials $a(X),b(X),c(X)\in \mathbb{F}_{<n+2}[x]$:

$$
\begin{aligned}
a(X)&=(b_1(X)+b_2)Z_H(X)+\sum_{i=1}^{n}w_{Li}L_i(X)
\\
b(X)&=(b_3(X)+b_4)Z_H(X)+\sum_{i=1}^{n}w_{Ri}L_i(X)
\\
c(X)&=(b_5(X)+b_6)Z_H(X)+\sum_{i=1}^{n}w_{Oi}L_i(X)
\end{aligned}
$$

- Compute $[a(x)]_1$, $[b(x)]_1$, $[c(x)]_1$. Append them into `transcript`.

#### Round 2

- Compute the table polynomial $t(X)\in \mathbb{F}_m[x]$:

$$
t(X)=\sum_{i=1}^{m}t_iL_i(X)
$$

- Let $\mathbf{s}\in \mathbb{F}^{2n}$ be the vector that is $(\mathbf{f},\mathbf{t})$ sorted by $\mathbf{t}$. We represents $s$ by the vectors $\mathbf{h_1},\mathbf{h_2}\in \mathbb{F}^n$ as follows:

$$
\begin{aligned}
\mathbf{h_1}&=(s_1,s_3,...,s_{2n-1})
\\
\mathbf{h_2}&=(s_2,s_4,...,s_{2n})
\end{aligned}
$$

- Generate random blinding scalars $b_{7},...,b_{11}$.

- Compute the polynomials $h_1(X)\in \mathbb{F}_{<n+3}[x]$, and $h_2(X)\in \mathbb{F}_{<n+2}[x]$.

$$
\begin{aligned}
h_1(X)&=(b_7X^2+b_8X+b_9)Z_H(X)+\sum_{i=1}^{n}s_{2i-1}L_i(X)
\\
h_2(X)&=(b_{10}X^2+b_{11}X)Z_H(X)+\sum_{i=1}^{n}s_{2i}L_i(X)
\end{aligned}
$$

- Compute $[t(x)]_1$, $[h_1(x)]_1$, $[h_2(x)]_1$. Append them into `transcript`.

#### Round 3

- Compute the permutation challenges $\beta,\gamma,\delta,\varepsilon\in \mathbb{F}$:

$$
\beta=\mathrm{Hash(transcript \ | \ 1)}
\\
\gamma=\mathrm{Hash(transcript \ | \ 2)}
\\
\delta=\mathrm{Hash(transcript \ | \ 3)}
\\
\varepsilon=\mathrm{Hash(transcript \ | \ 4)}
$$

- Generate random blinding scalars $b_{12},...,b_{17}$.
- Compute the Plonk permutation polynomial $z_1(X)\in \mathbb{F}_{<n+3}[x]$:

$$
\begin{aligned}
z_1(X)=\ &(b_{12}X^2+b_{13}X+b_{14})Z_H(X)+L_1(X)
\\
&+\sum_{i=1}^{n-1}\left(L_{i+1}(X)\prod_{j=1}^i \frac{(w_{Lj}+\beta\omega^{j-1}+\gamma)(w_{Rj}+\beta k_1\omega^{j-1}+\gamma)(w_{Oj}+\beta k_2\omega^{j-1}+\gamma)}{(w_{Lj}+\beta\sigma_{1j}+\gamma)(w_{Rj}+\beta\sigma_{2j}+\gamma)(w_{Oj}+\beta\sigma_{3j}+\gamma)} \right)
\end{aligned}
$$

- Compute the plookup permutation polynomial $z_2(X)\in \mathbb{F}_{<n+3}[x]$:

$$
\begin{aligned}
z_2(X)=\ &(b_{15}X^2+b_{16}X+b_{17})Z_H(X)+L_1(X)
\\
&+\sum_{i=1}^{n-1}\left(L_{i+1}(X)\prod_{j=1}^{i}\frac{(1+\delta)(\varepsilon+q_{Kj}c_j)(\varepsilon(1+\delta)+t_j+\delta t_{j+1})}{(\varepsilon(1+\delta)+s_{2j-1}+\delta s_{2j})(\varepsilon(1+\delta)+s_{2j}+\delta s_{2j+1})}\right)
\end{aligned}
$$

- Compute $[z_1(x)]_1$ and $[z_2(x)]_1$.

#### Round 4

- Compute the quotient challenge $\alpha\in \mathbb{F}$:

$$
\alpha=\mathrm{Hash(transcript)}
$$

- Compute the quotient polynomial $q(X)\in \mathbb{F}_{<3n+5}[x]$:

$$
q(X)=\frac{1}{Z_H(X)}
\left(
\begin{aligned}
&\ a(X)b(X)q_M(X)+q(X)q_L(X)+b(X)q_R(X)+c(X)q_O(X)+q_C(X)+\mathrm{PI}(X)
\\
&+(a(X)+\beta X+\gamma)(b(X)+\beta k_1X+\gamma)(c(X)+\beta k_2X+\gamma)z_1(X)\alpha
\\
&-(a(X)+\beta S_{\sigma 1}(X)+\gamma)(b(X)+\beta S_{\sigma 2}(X)+\gamma)(c(X)+\beta S_{\sigma 3}(X)+\gamma)z_1(\omega X)\alpha
\\
&+(z_1(X)-1)L_1(X)\alpha^2
\\
&+z_2(X)(1+\delta)(\varepsilon+q_K(X)c(X))(\varepsilon(1+\delta)+t(X)+\delta t(\omega X))\alpha^3
\\
&-z_2(\omega X)(\varepsilon(1+\delta)+h_1(X)+\delta h_2(X))(\varepsilon(1+\delta)+h_2(X)+\delta h_1(\omega X))\alpha^3
\\
&+(z_2(X)-1)L_1(X)\alpha^4
\\
&+q_T(X)t(X)\alpha^5
\end{aligned}
\right.
$$

- Split $q(X)$ into 3 polynomials $q_{lo}(X)$, $q_{mid}(X)$, $q_{hi}(X)$ of degree at most $n+1$ such that:

$$
q(X)=q_{lo}(X) + X^{n+2}q_{mid}(X)+X^{2n+4}q_{hi}(X)
$$

- Compute $[q_{lo}(x)]_1$, $[q_{mid}(x)]_1$, $[q_{hi}(x)]_1$. Append them into `transcript`.

#### Round 5

- Compute the evaluation challenge $\xi\in \mathbb{F}$:

$$
\xi=\mathrm{Hash(transcript)}
$$

- Compute the opening evaluations and append them into `transcript`:

$$
a(\xi),b(\xi),c(\xi),S_{\sigma_1}(\xi),S_{\sigma_2}(\xi),q_K(\xi),t(\xi),t(\omega \xi),
\\
z_1(\omega \xi),z_2(\omega \xi),h_1(\omega \xi),h_2(\xi)
$$

#### Round 6

- Compute the opening challenge $\eta\in \mathbb{F}$:

$$
\eta=\mathrm{Hash(transcript)}
$$

- Compute the linearization polynomial $r(X)\in \mathbb{F}_{<n+3}[x]$:

$$
\begin{aligned}
r(X) =&\ a(\xi)b(\xi)q_M(X)+a(\xi)q_L(X)+b(\xi)q_R(X)+c(\xi)q_O(X)+PI(\xi)+q_C(X)
\\
&+\alpha[(a(\xi)+\beta\xi+\gamma)(b(\xi)+\beta k_1\xi+\gamma)(c(\xi)+\beta k_2\xi+\gamma)z_1(X)
\\
&-(a(\xi)+\beta S_{\sigma 1}(\xi) + \gamma)(b(\xi)+\beta S_{\sigma 2}(\xi)+\gamma)(c(\xi)+\beta S_{\sigma 3}(X)+\gamma)z_1(\omega \xi)]
\\
&+\alpha^2(z_1(X)-1)L_1(\xi)
\\
&+\alpha^3[z_2(X)(1+\delta)(\varepsilon+q_K(\xi)c(\xi))(\varepsilon(1+\delta)+t(\xi)+\delta t(\omega \xi))
\\
&-z_2(\omega \xi)(\varepsilon (1+\delta)+h_1(X)+\delta h_2(\xi))(\varepsilon (1+\delta)+h_2(\xi)+\delta h_1(\omega\xi))]
\\
&+\alpha^4(z_2(X)-1)L_1(\xi)
\\
&+\alpha^5q_T(X)t(\xi)
\\
&-Z_H(\xi)(q_{lo}(X)+\xi^{n+2}q_{mid}(X)+\xi^{2n+4}q_{hi}(X))
\end{aligned}
$$

- Compute the opening proof polynomial $W_\xi(X) \in \mathbb{F}_{<n+2}[x]$

$$
W_{\xi}(X)=\frac{1}{X-\xi}
\left(
\begin{aligned}
&\ r(X)
\\
&+\eta(a(X)-a(\xi))
\\
&+\eta^2(b(X)-b(\xi))
\\
&+\eta^3(c(X)-c(\xi))
\\
&+\eta^4(S_{\sigma 1}(X)-S_{\sigma 1}(\xi))
\\
&+\eta^5(S_{\sigma 2}(X)-S_{\sigma 2}(\xi))
\\
&+\eta^6(q_K(X)-q_K(\xi))
\\
&+\eta^7(h_2(X)-h_2(\xi))
\\
&+\eta^8(t(X)-t(\xi))
\end{aligned}
\right)
$$

- Compute the opening proof polynomial $W_{\omega\xi}(X)\in \mathbb{F}_{<n+2}[X]$:

$$
W_{\omega\xi}(X)=\frac{1}{X-\omega\xi}
\left(
\begin{aligned}
&\ t(X)-t(\omega\xi)
\\
&+\eta(z_1(X)-z_1(\omega\xi))
\\
&+\eta^2(z_2(X)-z_2(\omega\xi))
\\
&+\eta^3(h_1(X)-h_1(\omega\xi))
\end{aligned}
\right)
$$

- Compute $[W_{\xi}(x)]_1$ and $[W_{\omega\xi}(x)]_1$.

Finally, we have the complete proof:

$$
\pi=\left(
\begin{matrix}
[a(x)]_1,[b(x)]_1,[c(x)]_1,
\\
[t(x)]_1,[h_1(x)]_1,[h_2(x)]_1,[z_1(x)]_1,[z_2(x)]_1,
\\
[q_{lo}(x)]_1,[q_{mid}(x)]_1,[q_{hi}(x)]_1,[W_{\xi}(x)]_1,[W_{\omega\xi}(x)]_1,
\\
a(\xi),b(\xi),c(\xi),S_{\sigma 1}(\xi),S_{\sigma 2}(\xi),q_K(\xi),t(\xi),t(\omega\xi),
\\
z_1(\omega\xi),z_2(\omega\xi),h_1(\omega\xi),h_2(\xi)
\end{matrix}
\right)
$$

### 5.3 Verify Process

- Validate all elements in $\pi$ are valid.
- Validate $(w_i)_{i<l}$ are valid.
- Compute all the challenges $\beta,\gamma,\delta,\varepsilon,\alpha,\xi,\eta$.
- Compute the vanishing polynomial evaluation $Z_H(\xi)=\xi^n-1$.
- Compute the first Lagrange polynomial evaluationn $L_1(\xi)=\frac{\xi^n-1}{n(\xi-1)}$
- Compute the public input polynomial evaluation $\mathrm{PI}(\xi)=\sum_{i=1}^{l}w_iL_i(\xi)=\sum_{i=1}^{l}\frac{w_i(\xi^n-1)\omega^{i-1}}{n(\xi-\omega^{i-1})}$.
- Compute the first evaluation $u$:

$$
\begin{aligned}
u=&\ \mathrm{PI}(\xi)-\alpha(a(\xi)+\beta S_{\sigma 1}(\xi) + \gamma)(b(\xi)+\beta S_{\sigma 2}(\xi)+\gamma)(c(\xi)+\gamma)z_1(\omega\xi)-\alpha^2L_1(\xi)
\\
&-\alpha^3z_2(\omega \xi)(\varepsilon (1+\delta)+\delta h_2(\xi))(\varepsilon (1+\delta)+h_2(\xi)+\delta h_1(\omega\xi))-\alpha^4L_1(\xi)
\\
&-\eta a(\xi)-\eta^2 b(\xi)-\eta^3 c(\xi)-\eta^4 S_{\sigma 1}(\xi)-\eta^5 S_{\sigma 2}(\xi)-\eta^6 q_K(\xi)-\eta^7 h_2(\xi)-\eta^8 t(\xi)
\end{aligned}
$$

- Compute the first polynomial commitment $[U]_1$:

$$
\begin{aligned}
[U]_1=&\ a(\xi)b(\xi)[q_M(x)]_1+a(\xi)[q_L(x)]_1+b(\xi)[q_R(x)]_1+c(\xi)[q_O(x)]_1+[q_C(x)]_1
\\
&+(\alpha(a(\xi)+\beta\xi+\gamma)(b(\xi)+\beta k_1\xi+\gamma)(c(\xi)+\beta k_2\xi+\gamma)+\alpha^2L_1(\xi))[z_1(x)]_1
\\
&-\alpha\beta z_1(\omega \xi)(a(\xi)+\beta S_{\sigma 1}(\xi) + \gamma)(b(\xi)+\beta S_{\sigma 2}(\xi)+\gamma)[S_{\sigma 3}(x)]_1
\\
&+(\alpha^3(1+\delta)(\varepsilon+q_K(\xi)c(\xi))(\varepsilon(1+\delta)+t(\xi)+\delta t(\omega \xi))+\alpha^4L_1(\xi))[z_2(x)]_1
\\
&+\alpha^5t(\xi)[q_T(x)]_1-Z_H(\xi)\left([q_{lo}(x)]_1+\xi^{n+2}[q_{mid}(x)]_1+\xi^{2n+4}[q_{hi}(x)]_1\right)
\\
&+\eta[a(x)]_1+\eta^2[b(x)]_1+\eta^3[c(x)]_1+\eta^4[S_{\sigma 1}(x)]_1+\eta^5[S_{\sigma 2}(x)]_1
\\
&+\eta^6[q_K(x)]_1+\eta^7[h_2(x)]_1+\eta^8[t(x)]_1
\end{aligned}
$$

- Compute the second evaluation $v$:

$$
v=-t(\omega\xi)-\eta z_1(\omega\xi)-\eta^2z_2(\omega\xi)-\eta^3h_1(\omega\xi)
$$

- Compute the second polynomial commitment $[V]_1$:

$$
[V]_1=[t(x)]_1+\eta[z_1(x)]_1+\eta^2[z_2(x)]_1+\eta^3[h_1(x)]_1
$$

- Verify 2 KZG opening proof by pairing engine $e([\bullet]_1,[\bullet]_2)$:

$$
\begin{aligned}
e([W_{\xi}(x)]_1,[x]_2)&\overset{?}{=}e([U]_1+u\cdot[1]_1+\xi\cdot[W_{\xi}(x)]_1,[1]_2)
\\
e([W_{\omega\xi}(x)]_1,[x]_2)&\overset{?}{=}e([V]_1+v\cdot[1]_1+\omega\xi\cdot[W_{\omega\xi}(x)]_1,[1]_2)
\end{aligned}
$$

- Verify $m$ KZG opening proof at each identifier $t_i$ by pairing engine $e([\bullet]_1,[\bullet]_2)$, where $1 \leq i \leq m$:

$$
e([f_i(x)]_1,[x]_2)\overset{?}{=}e([t(x)]_1-t_i\cdot[1]_1+\omega^{i-1}\cdot[f_i(x)]_1,[1]_2)
$$

## 5. ASP Part @Xxiang

association set providers(ASPs).

## 6. Client-Side Acceleration Solution


## References

[^1]:[plookup: A simplified polynomial protocol for lookup tables](https://eprint.iacr.org/2020/315.pdf)
[^2]:[plonkup: PlonKup: Reconciling PlonK with plookup](https://eprint.iacr.org/2022/086)
