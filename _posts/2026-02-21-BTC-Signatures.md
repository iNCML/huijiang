---
layout: distill
title: Elliptic Curve Digital Signatures in Bitcoin
description: In this post, we explain the cryptographic use of elliptic curves by first building the underlying finite abelian group structure, and then deriving ECDSA and Schnorr signatures, as well as the  multi-signature (MuSig) scheme, which are currently used in the Bitcoin network.
date: 2026-02-21
future: true
htmlwidgets: true
hidden: false
tags: btc, elliptic curve, digital signature, 

authors:
  - name: Hui Jiang
    url: "https://www.cse.yorku.ca/~huijiang"
    affiliations:
      name: York University, Toronto, Canada
  
# must be the exact same name as your blogpost
bibliography: 2026-02-21-BTC-Signatures.bib

# Add a table of contents to your post.
#   - make sure that TOC names match the actual section names
#     for hyperlinks within the post to work correctly. 
#   - please use this format rather than manually creating a markdown table of contents.
toc:
  - name: Elliptic Curves as Finite Groups
  - name: Elliptic Curve Digital Signatures 
    subsections:
    - name: ECDSA
    - name: Schnorr
    - name: ECDSA vs Schnorr
  - name: Multi-Signatures and MuSig
  - name: Concluding Remarks
---

# Elliptic Curve Digital Signatures in Bitcoin

> **Abstract** 
> We explain the cryptographic use of elliptic curves by first building the underlying *finite abelian group* structure, and then deriving **ECDSA** and **Schnorr** signatures along with the  multi-signature (multisig) scheme, which are currently used in the Bitcoin network. *See [Bitcoin Signature Lab](http://www.cse.yorku.ca/~huijiang/BTC/btc-signature-lab/) for an interactive demo to demenstrate all digital signature methods explained in this post.*


# 1. Elliptic Curves as Finite Groups

This section first explains the foundation model you need for ECDSA/Schnorr: **elliptic curves** are a way to construct a finite group in which a discrete logorithm problem is hard.

## 1.1 Curves over different domains

An elliptic curve is typically presented (short Weierstrass form) as

$$
E: \quad y^2 = x^3 + ax + b.
$$

What “points on the curve” means depends on the underlying set for $$ (x,y)$$:

- Over $\mathbb{R}$: infinitely many points; you can draw the familiar smooth curve.
- Over a finite field $\mathbb{F}_p$ (prime $p$): **only finitely many points**, hence a **finite group**.

In modern ECC (including Bitcoin), we almost always use a finite field $\mathbb{F}_p$ with large prime $p$.

## 1.2 Why $\mathbb{F}_p$ automatically gives finitely many points

In $\mathbb{F}_p$,

$$
\mathbb{F}_p = \{0,1,2,\dots,p-1\},
$$

so there are only $p$ possible $x$-values and $p$ possible $y$-values. Therefore, the set of candidate pairs $(x,y)$ is at most $p^2$, and the curve’s solutions are a subset.
Hence the size of $E(\mathbb{F}_p)$, called **order**, is finite.

More precisely, for each fixed $x$, the equation

$$
y^2 \equiv r \pmod p,\quad r \equiv x^3 + ax + b \pmod p
$$

has either $0$, $1$, or $2$ solutions for $y$. This already suggests the total number of points is on the order of $p$.

We require the curve to be **non-singular**, which  is equivalent to a nonzero discriminant:

$$
\Delta = -16(4a^3 + 27b^2) \not\equiv 0 \pmod p.
$$

This condition prevents cusps/self-intersections and ensures the group law behaves well (notably, associativity).

## 1.3 The “strange” addition law: geometry → algebra

Over $\mathbb{R}$, we define point addition via the line-through-two-points picture:

- Given $P$ and $Q$, draw the line $L$ through them.
- It intersects the cubic at a third point $R$ (counting multiplicity).
- Define $P + Q$ to be the reflection of $R$ across the $x$-axis.

In a finite field $\mathbb{F}_p$, you cannot literally “draw” a line—yet a similar rule is used instead according to **pure algebra**.    For example, write the line as

$$
y = mx + c,
$$

substitute it into the curve, and you obtain a cubic equation in $x$:

$$
(mx+c)^2 = x^3 + ax + b \quad \Longrightarrow \quad x^3 - m^2x^2 + (a-2mc)x + (b-c^2)=0.
$$

If $L$ passes through $P$ and $Q$, then $x_P$ and $x_Q$ are roots of this cubic, so there is a third root $x_R$ (in the algebraic closure). Moreover, because the coefficients and $x_P,x_Q$ lie in $\mathbb{F}_p$, Vieta’s relations imply

$$
x_P + x_Q + x_R = m^2 \in \mathbb{F}_p \quad \Longrightarrow \quad x_R \in \mathbb{F}_p.
$$

So the “third intersection point” is not a topological fact—it is a **polynomial fact**. Here, this third intersection point is defined as the result of adding $P$ to $Q$ in $\mathbb{F}_p$, i.e. $P+Q$.

## 1.4 Why this forms a group

With the above definitions, one can show $E(\mathbb{F}_p)$ is an **abelian group**:

- **Closure:** algebraic third-intersection argument.
- **Identity:** $O$, which is the **point at infinity**. As we will see later, we have the fact that $
P + O = O + P = P$, which makes  $O$ the identity.
- **Inverse:** the inverse of $P=(x,y)$ is defined as $ -P = (x,-y).$ As we will see later, we have $ P +(-P) = O$.
- **Commutativity:** $P+Q=Q+P$ (same line).

For cryptography, we rely on this group structure being correct and efficiently computable.


## 1.5 How to compute $P+Q$ explicitly in $\mathbb{F}_p$

Let the curve be in short Weierstrass form over a prime field $\mathbb{F}_p$:

$$
E: \quad y^2 \equiv x^3 + ax + b \pmod p,
$$

and let

$$
P=(x_1,y_1),\quad Q=(x_2,y_2)\in E(\mathbb{F}_p).
$$

All arithmetic below is performed **modulo $p$** (for coordinates).

### Case A: Add to Identity

In this case, we have $P = O$ or $Q = O$. Therefore, we define 

$$
P + O = P,\qquad O + Q = Q.
$$

### Case B: Vertical line 

In this case $x_1 = x_2$ and $y_1 \equiv -y_2 \pmod p$.
The line through $P$ and $Q$ is “vertical”, so the sum is the identity:

$$
P + Q = O.
$$

This is exactly the group-theoretic inverse rule, since $Q=-P$ in this case. Therefore, we have $ P - P = P +(-P) = O$.

### Case C: General Addition

In this case, $P \neq Q$. We compute the slope

$$
m \equiv \frac{y_2 - y_1}{x_2 - x_1} \pmod p
      \equiv (y_2 - y_1)\cdot (x_2 - x_1)^{-1} \pmod p.
$$

Then compute the third-intersection algebraically (the point where the line meets the curve “the third time”):

$$
x_3 \equiv m^2 - x_1 - x_2 \pmod p,
$$

$$
y_3 \equiv m(x_1 - x_3) - y_1 \pmod p.
$$

Define

$$
P+Q = (x_3,y_3).
$$

> **Note.** The “reflect across the $x$-axis” step is already built into the formula for $y_3$ above.

### Case D: Doubling

In this case, we have $P = Q$, i.e., $P + Q = 2P$.

If $y_1 \equiv 0 \pmod p$, then the tangent is vertical and

$$
2P = O.
$$

Otherwise compute the tangent slope:

$$
m \equiv \frac{3x_1^2 + a}{2y_1} \pmod p
      \equiv (3x_1^2 + a)\cdot (2y_1)^{-1} \pmod p,
$$

and then reuse the same coordinate formulas:

$$
x_3 \equiv m^2 - 2x_1 \pmod p,
$$

$$
y_3 \equiv m(x_1 - x_3) - y_1 \pmod p.
$$

So

$$
2P = (x_3,y_3).
$$

### Division in $\mathbb{F}_p$: computing the inverse $(\cdot)^{-1}$

The term $(x_2-x_1)^{-1}$ (or $(2y_1)^{-1}$) means the **multiplicative inverse mod $p$**.
Since $p$ is prime, every nonzero value has an inverse. Two standard ways:

- **Extended Euclid:** find $u,v$ such that $(x_2-x_1)u + pv = 1$, then $u \equiv (x_2-x_1)^{-1} \pmod p$.
- **Fermat:** for nonzero $t$,

  $$
  t^{-1} \equiv t^{p-2} \pmod p.
  $$

In real-world ECC implementations, inversions are relatively expensive; many libraries use projective coordinates (e.g., Jacobian coordinates) to replace most inversions with multiplications, doing a single inversion at the end.

## 1.6 From finite group to cryptography: scalar multiplication

Given a point $G$, define **scalar multiplication**:

$$
kG = \underbrace{G + G + \cdots + G}_{k\text{ times}}.
$$

Efficient algorithms (double-and-add, windowed methods) compute $kG$ in $O(\log k)$ group operations.

The core hardness assumption is that, given $G$ and $Q=dG$, computing $d$ is infeasible, which is called Elliptic Curve Discrete Logarithm Problem (**ECDLP**).

## 1.7 Bitcoin’s concrete curve: secp256k1

Bitcoin uses the curve secp256k1 over $\mathbb{F}_p$ with

$$
p = 2^{256} - 2^{32} - 977,
\qquad
E: y^2 \equiv x^3 + 7 \pmod p.
$$

A base point $G$ generates a subgroup of large prime order $n$. Private keys are scalars $d\in[1,n-1]$, public keys are $Q=dG$.

In the Bitcoin network, we choose $G = (x_G, y_G)$ with 

``x_G=0x79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798``
``y_G=0x483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8``

and the order $n$, i.e., the smallest positive integer such as $n G = O$ is 

``n =
FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141.``

> **Takeaway:** Elliptic curves on $\mathbb{F}_p$ give us a *finite* group with a hard discrete logarithm problem and fast group operations. Signatures (ECDSA, Schnorr) are built on top of this exact structure.

---

# 2. Elliptic Curve Digital Signatures

This section provides a rigorous and structured treatment of:

-   **ECDSA (Elliptic Curve Digital Signature Algorithm)**
-   **Schnorr Signatures (BIP340 / Taproot in Bitcoin)**

Both rely on the hardness of the **Elliptic Curve Discrete Logarithm Problem (ECDLP)**:

$$ \text{Given }G \text{ and } Q =
dG, \text{recover}  \; d $$

We assume a cyclic elliptic curve group of prime order $ n $ with generator $ G $.

Private key: $$ d  \in \{ 1, 2, \cdots, n-1\} $$

Public key: $$ Q = dG $$

All scalar arithmetic is performed modulo $ n $.

Due to the computational hardness of the Elliptic Curve Discrete Logarithm Problem (**ECDLP**), it is infeasible to recover the private key $d$ from the corresponding public key $Q$.  In a digital signature scheme, we assume that the holder of the private key—referred to as the signer—intends to authenticate a piece of information, called a **message**. The fundamental idea behind digital signatures is to design a secure signing algorithm in which the signer uses the private key to process the message and produce a value known as a **signature**, which is used to acknowledge the message. The signing procedure must be constructed so that no party can generate a valid signature without knowledge of the private key. The second phase is verification. In this phase, any party can execute a verification algorithm that uses only the public key, together with the **message** and the **signature**, to undo the processing in the signing stage to be able to check a specific deterministic condition. If the verification succeeds, it provides strong assurance that the signature was produced by the legitimate signer, since only the signer possesses the private key required to generate a valid signature.

---

## 2.1 ECDSA

### 2.1.1 Signing Algorithm: message + private key  →  signature 

Given a message $ m $:

##### Step 1: Hash the message

$$ e = H(m) $$

##### Step 2: Choose nonce

$$
\text{choose randomly:} \;\;\; \;\; k \in \{ 1, 2, \cdots, n-1\}
$$

##### Step 3: Compute an EC point

$$ 
R = kG = (x_R, y_R) 
$$ 

where choose $$ r = x_R \; \pmod{n} $$

##### Step 4: Compute signature 

$$ s = k^{-1}(e + rd) \; \text{mod} \; n $$

**Signature** is produced as $ (r, s). $

### 2.1.2 Verification: message + signature + public key  →  accept/reject

Given the signature $ (r,s) $ and the message $m$:

##### Step 1: Hash the message $m$

$$ 
e = H(m) 
$$ 

##### Step 2: process the message hash $e$ with signature $(r,s)$

$$ 
w = s^{-1}  \; \text{mod} \; n 
$$ 

$$ 
u_1 = ew \; \text{mod} \; n 
$$ 

$$ 
u_2 = rw  \; \text{mod} \;  n 
$$


##### Step 3: produce the result using the public key $Q$ 

$$ P = u_1 G + u_2 Q $$

##### Step 4: verification

Accept if: 

$$ 
x_P \equiv r \; \text{mod} \; n 
$$

### 2.1.3 Why It Works

From the signing formula: 
$$ s = k^{-1}(e + rd) $$

Rearrange: 
$$ k = s^{-1}(e + rd) $$

Verification undoes the signing process to reconstruct the same $R$: 

$$ P = e s^{-1} G + r s^{-1} Q = s^{-1}(e + rd)G = kG = R $$


### 2.1.4 ECDSA Diagram

    Signer: m + d → (r,s)

       m → e = H(m)
       k → R = kG
       r = x_R
       s = k⁻¹(e + r d)
       

    Verifier: m + (r,s) + Q → valid/not

       m → e = H(m)
       w = s⁻¹
       P = e w G + r w Q
       Check x_P = r


> **Note.** In ECDSA (including Bitcoin), if you have a valid signature $(r, s)$, then $(r, n - s)$ is also valid, where n is the secp256k1 curve order. So without a rule, one signature has two valid forms (high-S and low-S). 
Low-S normalization means:
• If $s > n/2$, replace it with $s = n - s$.
• Otherwise keep $s$ as is.
This forces a unique canonical form (the “lower half” value of $s$), which helps prevent signature malleability in Bitcoin policy/consensus contexts.

---

## 2.2 Schnorr Signature (BIP340)
 
### 2.2.1 Signing

Given message $ m $:

* Choose a random $ k  \in \{ 1,2, \cdots, n-1 \}$:

$$ 
R = kG 
$$

* Compute challenge: 

$$ 
e = H(R \| Q \| m) 
$$

* Compute response: 

$$ 
s = k + e \,d \; \pmod{n} 
$$

Signature is generated as $$ (R, s).$$


### 2.2.2 Verification

* Compute challenge in the same way as above: 

$$ 
e = H(R \| Q \| m) 
$$

* Compute:

$$ 
P  =  s G  - e Q 
$$


* Accept if: 

$$ 
x_P \equiv x_R \; \text{mod} \; n 
$$


### 2.2.3 Why It Works

From the signing formula: 

$$ s = k + ed $$

Multiply by $ G $:

$$ sG = kG + edG = R + eQ $$

$$ \implies P = s G  - e Q  = R $$


## 2.2.4 Why Do We Compute the Challenge e in Schnorr?

In the Schnorr signing algorithm, after computing the nonce point

$$
R = kG,
$$

we compute the **challenge** using a cryptographic hash function $$ H(\cdot) $$ as follows:

$$
e = H(R \,\|\, Q \,\|\, m),
$$

which binds everything together:

- $$ R $$ = nonce commitment  
- $$ Q $$ = public key  
- $$ m $$ = message  

This step is essential for **security and correctness**. Below is why.

First of all, if we did not hash the message into the signature equation, the signature could be reused for different messages. Because $$ e $$ depends on $$ m $$, changing the message changes $$ e $$, and therefore changes the final signature $$ s $$. This guarantees that signature validity is message-dependent.

Secondly, if the nonce commitment $$ R $$ is not included in the hash that defines the challenge $$ e $$, the Schnorr signature scheme becomes forgeable. Specifically, suppose the challenge is computed as $$ e = H( Q \,\|\, m) $$, which does not depend on $$ R $$. A forger can then choose an arbitrary scalar  $$ s' $$ and construct a forged signature  $$ (R', s') $$ by setting $$ R' = s' G - e Q $$. This pair will pass verification, because the verifier checks $$ s' G = R' + e Q $$, which is always true from the way to choose $$ R' $$. Thus, the forged signature satisfies the verification equation without requiring knowledge of the private key. In contrast, when the challenge is defined as $$ e = H( R \,\|\, Q \,\|\, m) $$, the scheme is secure against this attack. In this case, the challenge depends on $$ R $$, so a forger cannot freely choose $$ s' $$ and then compute a corresponding $$ R' $$. Any attempt to construct $$ R' $$ requires knowing $$ e $$ but $$ e $$ itself depends on $$ R' $$, creating a circular dependency. This circularity prevents the attacker from generating a valid signature without the private key.

Finally, if the challenge $$ e $$ is not bound to the public key $$ Q $$,  the scheme becomes vulnerable to a key-substitution (rogue-key) attack. Suppose the challenge is computed as  $$ e = H( R \,\|\, m) $$,  so it does not depend on the public key. Given a valid signature $$ (R,s) $$ on message $$ m $$ under public key $$ Q $$, an attacker can construct a different public key

$$
   Q' = \frac{s G - R }{e} = e^{-1} (s G - R )
$$

Now consider verification under this new public key $$ Q' $$. The verifier checks whether $$ P = R $$ holds. Substituting the definition of $$ Q' $$, the verification equation holds:

$$
P = sG - e Q' = sG  - e \cdot e^{-1} (s G - R ) = R
$$

Consequently, the same signature $$ (R,s) $$ is valid for the same message $$ m $$ under both the original public key  $$ Q $$ and the attacker-constructed public key $$ Q' $$.  In other words, anyone can take a genuine signature and fabricate a public key that makes the signature verify. This breaks the fundamental security property of digital signatures: the signature no longer proves that the holder of the original public key $$ Q $$ produced it. Binding the challenge to the public key, i.e., $$ e = H( R \,\|\, Q \,\|\, m) $$ prevents this attack, because the challenge would change if the public key were altered.


####   It Preserves Linearity (Crucial for MuSig)

The form

$$
s = k + ed
$$

is linear in the secret key $$ d $$.

Since $$ e $$ is a scalar derived from a hash, it preserves linear structure.  
This linearity enables:

- Key aggregation  
- Signature aggregation  
- MuSig multi-signatures  

>**Takeaway:** The computation of $$ e = H(R \,\|\, P \,\|\, m) $$ is what transforms Schnorr from a simple linear equation into a secure digital signature scheme.


------------------------------------------------------------------------

### 2.2.4 Schnorr Diagram

    Signer: m + d → (R,s)

       k → R = kG
       e = H(R || Q || m)
       s = k + e d


    Verifier: m + (R,s) + Q → valid/not

       Compute e = H(R || Q || m)
       compute P = sG - e Q
       Check P = R

------------------------------------------------------------------------

## 2.3 ECDSA vs Schnorr

| Feature             | ECDSA                          | Schnorr        |
|---------------------|--------------------------------|----------------|
| Formula             | $ s = k^{-1}(e + rd) $       | $ s = k + ed $ |
| Linear              | No                             | Yes            |
| Multi-signature     | Difficult                      | Natural        |
| Aggregation         | No                             | Yes            |
| Used in Bitcoin     | Legacy                         | Taproot        |

------------------------------------------------------------------------

# 3. Multi-Signatures (MuSig)

A multi-signature (MuSig) scheme allows multiple parties to jointly authorize a message by requiring several independent signatures, typically under a policy such as n-of-n (e.g., all n signers must approve). Each participant signs the same message with their own private key, and the combined signatures are verified against the corresponding public keys.  MuSig (a Schnorr-based multisignature scheme) improves upon traditional multisig by aggregating multiple public keys into a single combined public key and producing one compact signature that is indistinguishable from a standard single-party Schnorr signature. The signers interact to generate nonces and partial signatures, which are then combined into one final signature. Verification requires only the aggregated public key, enhancing privacy, efficiency, and scalability compared to classical multisig constructions.  It is simple to implement multisig using Schnorr Because Schnorr is linear.

Let two signers:

$$ Q_1 = d_1 G, \quad  Q_2 = d_2 G $$

Aggregate public key: 

$$ Q = Q_1 + Q_2 $$

Aggregate nonce: 

$$ R = R_1 + R_2 $$

Aggregate signature: 

$$ s = s_1 + s_2 $$

Verification can be done as usual:

$$ P = sG - e Q  \overset{?}{=} R $$

This property enables a simple 2-of-2 **MuSig** scheme, used in Bitcoin for n-of-n multi-party signing.

------------------------------------------------------------------------

# 4. Concluding Remarks

ECDSA remains widely deployed, but Schnorr offers:

-   Simpler algebra
-   Clean linear structure
-   Aggregation capability
-   Better provable security properties

Schnorr is considered the more modern and elegant construction.

------------------------------------------------------------------------
