---
layout: distill
title: The Bitcoin Lightning Network
description: This post presents a technical but accessible introduction to Lightning, starting from Bitcoin transaction primitives and building upward to channel construction, state updates, watchtowers, revocation, HTLC routing.
date: 2026-03-07
future: true
htmlwidgets: true
hidden: false
tags: btc, ligtning, revocation,  

authors:
  - name: Hui Jiang
    url: "https://www.cse.yorku.ca/~huijiang"
    affiliations:
      name: York University, Toronto, Canada
  
# must be the exact same name as your blogpost
bibliography: 2026-03-07-Lightning-Network.bib

# Add a table of contents to your post.
#   - make sure that TOC names match the actual section names
#     for hyperlinks within the post to work correctly. 
#   - please use this format rather than manually creating a markdown table of contents.
toc:
  - name: Introduction
  - name: The Lightning Protocol
    subsections:
    - name: Payment Channels as Off-Chain Shared Ledgers
    - name: Funding the Channel with 2-of-2 Multisig and P2WSH
    - name: Commitment Transactions
  - name: A Channel Example to Demenstrate Technical Detals
    subsections:
    - name: How to Open and Fund the Channel 
    - name: The Commitment Update Protocol 
    - name: Force Close vs Cooperative Close
    - name: What does Bitcoin actually see?
    - name: Why Watchtowers are needed
  - name: Revocation 
  - name: Shachain
  - name: HTLCs
  - name: Practical Advantages of Lightning
  - name: Conclusion
---


>**Abstract**
>
>Bitcoin’s base layer is optimized for decentralization, auditability, and adversarial robustness, not for high-throughput retail payments. The Lightning Network addresses this limitation by introducing a second-layer protocol built from bidirectional payment channels, revocable commitment states, hashed timelock contracts, and on-chain dispute resolution. This article presents a technical but accessible introduction to Lightning, starting from Bitcoin transaction primitives and building upward to channel construction, state updates, watchtowers, revocation, HTLC routing.


## 1. Introduction: Why Bitcoin Needs a Payment Layer

Bitcoin is exceptionally strong as a settlement system. Its design prioritizes consensus safety, resistance to censorship, verifiability, and the ability for ordinary users to audit the ledger. Those strengths come with well-known trade-offs. Blocks arrive roughly every ten minutes, throughput is limited, and every fully validated transaction must eventually be processed by the entire network. That trade-off is acceptable for high-value settlement, but it is awkward for small, frequent, interactive payments. Buying coffee, streaming micropayments per second, or routing tiny global transfers through the base chain alone would be slow and expensive.

The Lightning Network was developed to solve this problem without changing Bitcoin’s role as the final source of truth. The basic idea is elegant: instead of recording every payment on-chain, two users lock funds into a shared Bitcoin output, then privately update who owns what. Only the opening and closing of that relationship require on-chain transactions under normal operation. Everything in between is handled off-chain, but remains enforceable through Bitcoin scripts and signatures.

Lightning is therefore not a separate currency, not a sidechain, and not a custodial overlay. It is a contract-based payment system that uses Bitcoin as its court of final appeal. At a high level, Lightning is a network of payment channels. A channel connects two peers, allowing them to rebalance ownership of locked funds many times without repeatedly touching the blockchain.

If Alice and Bob share a channel, they can pay each other directly. If Alice does not share a direct channel with Bob, but Alice has a channel with Carol and Carol has one with Dave and Dave has one with Bob, then Alice can still pay Bob through a chain of conditional payments:

```text
Alice → Carol → Dave → Bob
```

This gives Lightning two scaling properties at once. First, it compresses many bilateral updates into a small number of on-chain transactions. Second, it allows a network of local channels to approximate global reach through routing.

The protocol achieves this while preserving several strong goals:

- either channel party can unilaterally exit to Bitcoin main chain
- outdated states are punishable
- routed payments are atomic
- intermediaries do not need custody over user funds
- final settlement remains in BTC on Bitcoin main chain

Those guarantees come from a combination of multisignature funding outputs, commitment transactions, revocation keys, timelocks, and HTLCs.

---

## 2. The Lightning Protocol

### 2.1 Payment Channels as Off-Chain Shared Ledgers

A Lightning channel begins with an on-chain funding transaction. Suppose Alice and Bob each contribute 1 BTC. The total channel capacity is therefore 2 BTC.

Economically, it is useful to imagine the channel as a tiny private ledger:

- Alice initially owns 1.0 BTC of the 2.0 BTC channel
- Bob initially owns 1.0 BTC of the 2.0 BTC channel

But that split is not represented directly on-chain. On-chain, there is only one shared output. The internal ownership split is represented by off-chain **commitment transactions**.

This distinction is central to Lightning:

- **on-chain reality:** one shared UTXO
- **off-chain reality:** a sequence of signed balance states

The entire protocol is about safely moving between those off-chain states.

### 2.2 Funding the Channel: 2-of-2 Multisig and P2WSH

At first, Alice and Bob need to open a lightning channel cooperatively by exchanging their public keys. The funding output must ensure that neither Alice nor Bob can spend the channel alone. The simplest construction is a 2-of-2 multisignature script:

```text
OP_2 <Alice_pubkey> <Bob_pubkey> OP_2 OP_CHECKMULTISIG
```

In Lightning, this script is typically committed via **P2WSH** (Pay-to-Witness-Script-Hash). The full witness script is hashed with SHA256, and the resulting output script is:

```text
OP_0 <32-byte SHA256(script)>
```

Alice and Bob need each contribute 1 BTC to the P2WSH address, thereby funding the channel with a total of 2 BTC.

This construction gives Lightning two crucial properties:

1. the channel capacity is secured by Bitcoin consensus
2. the channel can only move if the spending conditions encoded by the protocol are satisfied

#### P2WSH in Context

Because Lightning relies heavily on P2WSH, it is worth restating how it works. A P2WSH output commits to the SHA256 hash of a witness script. The actual script is not revealed until spend time. This is particularly useful for Lightning because commitment and HTLC scripts can be large and complex. By storing only the script hash on-chain at funding time, the protocol keeps opening transactions compact. It is also useful to distinguish P2WSH from P2WPKH. Both are version-0 SegWit outputs and both may begin with `bc1q...` as human-readable addresses, but on-chain they are distinguishable by witness program length:

- `0014 <20 bytes>` means P2WPKH
- `0020 <32 bytes>` means P2WSH


### 2.3 Commitment Transactions: The Heart of Lightning

Once the funding output exists, Alice and Bob need a way to represent who owns how much of it at any given moment. This is done with commitment transactions.

A commitment transaction spends the funding output and redistributes the funds according to the latest agreed channel state. Importantly, there are always **two** commitment transactions:

- Alice’s commitment transaction
- Bob’s commitment transaction

These are not identical byte-for-byte. They represent the same balances, but from different perspectives.

For example, if the current channel state is:

```text
Alice = 1.0 BTC
Bob   = 1.0 BTC
```

then Alice’s version might pay:

- `to_local` → Alice, delayed
- `to_remote` → Bob, immediate

while Bob’s version pays:

- `to_local` → Bob, delayed
- `to_remote` → Alice, immediate

The names are relative. “Local” means “me, the owner of this commitment transaction.” “Remote” means “the counterparty.” 

#### `to_local` and `to_remote` Outputs

The `to_remote` output is conceptually simple. It usually pays the counterparty in a straightforward SegWit form, often P2WPKH, and is immediately spendable.

The `to_local` output is the interesting one. It includes both a normal delayed withdrawal path and an immediate revocation path. The redeem condition for  `to_local` output  is typically designed as two deperate paths:

- **revocation path**: the counterparty may spend immediately if they can provide a valid signature under the revocation key. The signature requires the revocation secret that is not available before revocation.
- **delayed local path**: the owner of that commitment may spend, but only with their own key and only after waiting the required delay (144-block delay).

That waiting window is what gives the honest counterparty time to detect and punish a revoked-state broadcast (discuss later).

For example, the logic is roughly:

```text
IF revocation is used, check signature under revocation key
ELSE enforce delay, then check signature under delayed key
```

#### Relative Timelocks and the 144-Block Delay

Lightning primarily uses **relative timelocks** for `to_local` outputs. These are implemented using `OP_CHECKSEQUENCEVERIFY` (CSV) together with the spending input’s `nSequence`. A common tutorial value is 144 blocks, roughly one day. The key idea is that the owner of a `to_local` output cannot spend it immediately. They must wait relative to the confirmation height of the commitment transaction. If the commitment confirms at block 900000 and the delay is 144, then the earliest block where the delayed spend becomes valid is approximately 900144.

The enforcement is split across script and consensus:

- the script requires a minimum delay via `OP_CHECKSEQUENCEVERIFY`
- Bitcoin nodes interpret the spending input’s `nSequence` according to BIP68 and verify that enough blocks have elapsed

This may look slightly redundant, but it is a deliberate design: the script expresses the contract requirement, while the transaction metadata carries the declared relative locktime that nodes can enforce.

#### Why There Are Two Asymmetric Commitment Transactions Instead of One

A naïve design might try to use one shared transaction for both parties. That would fail to give each side an immediately usable unilateral exit and would make punishment logic ambiguous.

Lightning instead arranges that each party always holds a fully signed transaction they can broadcast on their own. To achieve that, each party signs the other party’s commitment transaction. In practical terms:

- Bob signs Alice’s commitment transaction
- Alice signs Bob’s commitment transaction

So at any moment:

- Alice holds a valid transaction she can broadcast if Bob disappears
- Bob holds a valid transaction he can broadcast if Alice disappears

This design allows unilateral closes without cooperation, while still making old states punishable (discuss later).

---

## 3. A Channel Example to Demenstrate Technical Detals

Consider a simple channel between Alice and Bob:

- Alice contributes 1 BTC
- Bob contributes 1 BTC
- total channel capacity = 2 BTC

Initial state:

```text
S0:
Alice = 1.0
Bob   = 1.0
```

Now imagine three off-chain payments:

1. Alice pays Bob 0.2 BTC  
2. Alice pays Bob 0.1 BTC  
3. Bob pays Alice 0.5 BTC  

The resulting state sequence is:

```text
S0: Alice 1.0, Bob 1.0
S1: Alice 0.8, Bob 1.2
S2: Alice 0.7, Bob 1.3
S3: Alice 1.2, Bob 0.8
```

Throughout this entire sequence, the blockchain still sees only the funding output. None of these intermediate balance changes are published on-chain. They exist only as evolving commitment transactions and revocation records held by Alice and Bob.

If they later cooperate to close the channel, they can produce a simple closing transaction on the Bitcoin chain paying:

- 1.2 BTC to Alice’s chosen address
- 0.8 BTC to Bob’s chosen address

### 3.1 How to Open and Fund the Channel (Set up S0)

Alice and Bob follow a typical protocol flow to work cooperatively to send funds to the channel's locked address and simultaneously set the channel state as `S0`. Assume Alice initiates the process. 

**Step 1**: Alice sends `open_channel`

**Step 2**: Bob replies `accept_channel`

**Step 3**: Both exchange inputs and pubkeys

**Step 4**: Both build the funding transaction that has two inputs and one output, without signatures at this stage

>**Inputs**: 
>  * Alice UTXO (1 BTC)
>  * Bob   UTXO (1 BTC)
>
>**Output**:
>  * 2 BTC → P2WSH multisig (channel)

**Step 5**: Compute *txid* for the above funding transaction. (It is possible to do so in SegWit without using inputs' signatures)

**Step 6**:  Construct two asymmetric commitment transactions referencing funding_txid

>* Alice commitment transaction: `S0_A`
>
>    **Input**:
>    *   P2WSH funding_output (2 BTC)
>
>    **Outputs**:
>    *  to_local: 1 BTC  → Alice (delayed)
>    * to_remote: 1 BTC → Bob   (immediate)

The `to_local` output usually includes a revocation clause that allows the counterparty (Bob) to seize all funds in the channel if Bob can provides a valid signature under `<revocation_pubkey>`, which is calculated for `S0_A`. Bob does not have the secret key to generate the valid signature at this stage. 

```
OP_IF
    <revocation_pubkey>
OP_ELSE
    144 OP_CHECKSEQUENCEVERIFY
    OP_DROP
    <Alice_delayed_pubkey>
OP_ENDIF
OP_CHECKSIG
```

The `to_remote` output uses Bob's P2WPKH address, where Bob can spend it immediately
```
OP_0 HASH160(Bob_pubkey)
```

>*  Bob's commitment transaction: `S0_B`
>
>   **Input**:
>    *   P2WSH funding_output (2 BTC)
>
>    **Outputs**:
>    * to_local:  1 BTC  → Bob (delayed)
>    * to_remote: 1 BTC  → Alice   (immediate)

Similarly, the `to_local` also includes a revocation clause that allows Alice to seize all funds if she can provide a valid signature. The `to_remote` output goes to Alice's P2WPKH address. 

 **Step 6**: Sign two commitment transactions first and exchange signatures and both verify 

**Step 7**: Both sign their own inputs in the funding transaction and exchange signatures, both verify

**Step 8**: Broadcast the funding transaction onto the Bitcoin network. After this funding transaction is confirmed, the channel is in state `S0`.


### 3.2 The Commitment Update Protocol (S0 → S1)

When Alice and Bob proceed from state  `S0` to `S1`, the protocol must guarantee that no party can exploit a race condition. A simplified update flow works as follows:

**Step 1**: The sender (Alice) proposes the new balance state: `S1: Alice 0.8, Bob 1.2`, and sends Bob a message `update balance`

**Step 2**: Both sides build new commitment transactions: 

>* Alice commitment transaction: `S1_A`
>
>    **Input**:
>    *  P2WSH funding_output (2 BTC)
>
>    **Outputs**:
>    * to_local: 0.8 BTC  → Alice (delayed)
>    * to_remote: 1.2 BTC  → Bob ( immediate)


>* Bob commitment transaction: `S1_B`
>
>    **Input**:
>    *  P2WSH funding_output (2 BTC)
>
>    **Outputs**:
>    * to_local: 1.2 BTC  → Bob (delayed)
>    * to_remote: 0.8 BTC  → Alice ( immediate)


The `to_local` outputs includes new revocation clauses that allows the counterparty to seize all funds under new revocation pubkeys, which are calculated for `S1`.

**Step 3**: The sender (Alice) signs the receiver’s (Bob) new commitment transaction `S1_B`, and send Bob the signed comitment transaction `S1_B_signed`.

**Step 4**: The receiver (Bob) revokes the old state `S0` by releasing the revocation secret of the last commitement transaction `S0_B`, which allows Alice to construct a valid revocation signature in `S0_B`. (From now on, if Bob broadcasts `S0_B`, Alice can seize entire channel.)

**Step 5**: The receiver (Bob) signs the sender’s (Alice) new commitment transaction  `S1_A`, and send Alice the signed commitment tx `S1_A_signed`.

**Step 6**: The sender (Alice) revokes the old state `S0` by releasing the revocation secret of the last commitement transaction `S0_A`, which allows Bob to construct a valid revocation signature in `S0_A`. (From now on, if Alice broadcasts `S0_A`, Bob can seize entire channel.)

**Step 7**: State `S1` becomes the current state of the channel. (All previous states become obsolete, and neither party has any incentive to broadcast them, since doing so could lead to a loss of funds.)

The critical invariant is:

```text
new state becomes safe before old state is revoked
```

This is what prevents a party from being trapped in a window where the old state is invalid but the new state is not yet spendable.

#### Why the protocol is safe

In this case, Alice is paying Bob and wants to move from S0 to S1.

- Bob first sends Alice the signature needed for **Alice’s** S1 commitment transaction
- only after Alice already has a safe S1 close does she reveal the revocation secret for S0
- then Alice sends Bob the signature for **Bob’s** S1 commitment transaction

So if someone tries to broadcast S0 *before* revocation, S0 was still valid. No theft occurs. If someone broadcasts S0 *after* revocation, the counterparty can punish. There is no unsafe middle ground.

### 3.3 Force Close vs Cooperative Close

A Lightning channel can end in two main ways.

#### Cooperative close

In a cooperative close, both parties agree on the final balance and construct a simple closing transaction spending the funding output directly to their chosen destination addresses. This is the preferred path because it avoids delay scripts and reduces fees.

In our running example, after reaching `S3`:

```text
Alice = 1.2 BTC
Bob   = 0.8 BTC
```

In a cooperative close, both Alice and Bob work together to construct a final closing transaction:

>**Input**:
>*  funding_output (2 BTC)
>
>**Outputs**:
>* to Alice's P2WPKH: 1.2 BTC 
>* to Bob's P2WPKH: 0.8 BTC  

After it is signed with all valid signatures, and then broadcast to the Bitcoin network.

#### Force close

If one party disappears or cooperation fails, the other can unilaterally broadcast its latest commitment transaction. In that case:

- the counterparty’s `to_remote` output is usually immediately spendable
- the broadcaster’s `to_local` output is delayed

This asymmetry is intentional. It ensures that if the broadcaster used an outdated state, the honest party has time to punish.

### 3.4 What does Bitcoin actually see?

In our running channel example, the Bitcoin chain normally sees only:

1. the funding transaction creating the multisig P2WSH channel output  
2. the closing transaction redistributing the final balances  

All balance updates, revocation exchanges, and routed forwarding logic remain off-chain unless there is a dispute or force close. This means that Bitcoin does not record all Lightning payments. It records only the opening and final settlement, unless a problem forces an intermediate state to touch the chain.


### 3.5 Why Watchtowers are needed

Lightning security assumes someone is watching the blockchain for revoked-state broadcasts. But users are not always online. Mobile wallets sleep, laptops disconnect, and noncustodial users may not continuously monitor the chain. Watchtowers are services that monitor on behalf of users. The basic idea is:

1. the user prepares encrypted penalty data for each revocable state  
2. the watchtower stores this data without learning the channel details  
3. the watchtower scans the chain for matching breach hints  
4. if a revoked commitment transaction appears, the tower broadcasts the penalty transaction

In effect, the watchtower acts as an outsourced alarm bell and responder. It does not need custody of the funds, and well-designed watchtower schemes preserve privacy by storing only encrypted blobs and small identifiers.

In a cooperative close, the tower never needs to act. But its mere existence makes offline use safer.

See the [Lightning Channel Explorer](https://incml.github.io/BTC-Explorer/btc-lightning-channel.html) for an interactive, step-by-step walkthrough of this example, with all the technical details explained in the context of the Lightning Network.

---

## 4 Revocation: How Cheating Is Punished

Lightning’s core security idea is simple:

> publishing an old state should be catastrophic for the cheater.

If Alice broadcasts an outdated commitment transaction after it has been revoked, Bob should be able to take the entire channel balance, not merely restore the fair split. This severe penalty makes cheating economically irrational.

The mechanism works because each old state has its own revocation key. When state `S1` is superseded by `S2`, the necessary secret information is released so that the counterparty can derive the revocation private key corresponding to `S1`.

Thus:

- before revocation, an old state may still be valid
- after revocation, that same old state becomes a trap

This is why Lightning can safely keep many obsolete commitment transactions in history without trusting either party to “forget” them.

If `S1` is later broadcast on-chain by Alice, Bob does the following:

1.  Detect the old commitment transaction `S1` on-chain.
2.	Locate the spendable output that contains the revocation/delay script.
3.	Construct a penalty transaction spending that output.
4.	In the witness or script input, they choose the revocation OP_IF  branch of the script.
5.	They provide a valid signature using the revocation secret.
6.	Because the revocation branch has no delay, the output can be spent immediately.

#### Per-Commitment Secrets and Revocation Key Derivation

In Lightning, each commitment state $i$ has a fresh per-commitment secret $s_i$ and corresponding public point $P_i = s_i G$. Each party also has a long-lived revocation base secret b with public basepoint $B = b G$. For each state, these two public points are combined to form a revocation public key

$$
R_i = B \cdot H(P_i \| B) + P_i \cdot H(B \| P_i)
$$

which can be placed into the commitment script immediately, because it depends only on public data. The matching private key is

$$
r_i = b \cdot H(P_i \| B) + s_i \cdot H(B \| P_i)
$$

but this cannot be computed at first, since it requires both the long-lived revocation base secret and the state-specific per-commitment secret. The missing per-commitment secret is disclosed only when that state is later revoked. As a result, the revocation path is impossible to use too early, yet becomes fully spendable exactly when the state becomes obsolete. This is what allows Lightning to pre-commit a penalty key on-chain while activating it only after revocation.

---

## 5. Shachain: Storing Many Revocation Secrets Efficiently

A long-lived channel may update thousands or millions of times. Naïvely storing one fresh revocation secret per state would be wasteful. Lightning solves this with **shachain**, a compact hash-based structure that allows a large sequence of revocation secrets to be represented with only O(log N) storage. In practice, for a 48-bit state space, a node needs to store at most about 48 secrets rather than millions. Conceptually, secrets are organized in a binary derivation structure. As newer secrets are received, older ones can often be derived and no longer need to be stored individually.

Real shachain is more subtle than a simple linear hash chain, but the key intuition is the same: the protocol exploits structure in the state numbering so that memory use stays tiny even for huge channels. Shachain matters because Lightning’s revocation system only works well if implementations can efficiently manage historical state.

---

## 6. HTLCs: Extending Channels into a Network

Channels alone only solve direct bilateral payments. To build a network, Lightning uses **Hashed Timelock Contracts** (HTLCs).

An HTLC combines two ideas:

- a **hashlock**: funds can be claimed only by revealing a secret preimage
- a **timelock**: if the secret is not revealed in time, the sender can refund

A conceptual HTLC says:

```text
If SHA256(secret) = H, pay the receiver.
Otherwise, after timeout, refund the sender.
```

This allows value to move across several channels without requiring trust in intermediaries.


#### Multi-Hop Payment Example

Suppose Alice wants to pay Bob 0.1 BTC through Carol and Dave:

```text
Alice → Carol → Dave → Bob
```

First, Bob chooses a random secret value `s` and computes its hash

```text
H = SHA256(s)
```

Bob sends only the hash `H` back to Alice, while keeping the actual secret s private.

Alice then offers Carol an HTLC that says “Carol may take 0.1 BTC if she can provide a value `s` such that `SHA256(s)=H` before a certain deadline. Otherwise, after the deadline, Alice gets the money back.” So this HTLC has two spending paths:
- **Success path**: Carol presents the correct preimage `s` before timeout
- **Timeout path**: after expiry, Alice reclaims the funds

This HTLC is recorded off-chain by Alice and Carol to update their channel commitment state. Before the HTLC, maybe the balance is:
- Alice: 1.0 BTC
- Carol: 1.0 BTC

After inserting the HTLC for 0.1 BTC, the state is updated as:
1. Alice: 0.9 BTC immediately spendable
2. Carol: 1.0 BTC immediately spendable
3. plus one pending HTLC output of 0.1 BTC, claimable by Carol with preimage `s`, otherwise refundable to Alice after timeout

A simplified pseudocode for the above third HTLC branch is
```text
IF
    Carol provides secret s
    AND SHA256(s) == H
    AND Carol's signature is valid
THEN
    Carol can spend
ELSE
    after timeout 
    Alice with her signature can spend
ENDIF
```



Carol does the same with Dave, and Dave does the same with Bob. Thus, each hop sets up a conditional payment tied to the same hash `H`, but with different timeout values.

At this point, Bob is the only party who knows the secret `s`. Therefore, only Bob can satisfy the final HTLC from Dave. When Bob claims that HTLC, he must reveal s. Once Dave sees s, he can use the same secret to claim his incoming HTLC from Carol. Carol then learns s and uses it to claim from Alice.

So the mechanism has a very elegant structure:
- the conditional payment commitments are created from Alice forward to Bob,
- but the secret `s` is revealed and propagates backward from Bob to Alice.

This is what makes the payment effectively atomic across the whole route. Bob cannot receive the money without revealing `s`, and once `s` is revealed, each upstream hop can use it to collect its own incoming payment. If Bob never reveals the secret, then nobody downstream can complete the claim, and each HTLC eventually expires and refunds safely.

The timeouts are deliberately staggered. For example, Alice’s HTLC to Carol expires later than Carol’s HTLC to Dave, which expires later than Dave’s HTLC to Bob. This gives each intermediary a safety window: after learning `s` from the downstream hop, it still has enough time to redeem its upstream HTLC before that one expires. Without staggered timeouts, an intermediary could be exposed to loss by paying downstream but not having enough time to collect upstream.

In summary, HTLC routing lets strangers forward payments without trusting one another: either the secret is revealed and everyone along the route can claim in order, or the secret never appears and the contracts simply time out and refund.

---

## 7. Practical Advantages of Lightning

Lightning’s security is not based on preventing cheating in an absolute sense. Instead, it is based on making cheating detectable and catastrophically expensive.

The security stack includes:

- the funded 2-of-2 output anchoring the channel on Bitcoin
- commitment transactions giving each side a unilateral exit
- relative timelocks delaying the broadcaster’s own funds
- revocation keys making obsolete states punishable
- HTLC timeouts preserving atomicity in routed payments
- watchtowers assisting offline monitoring

The key economic rule is:

> honest behavior yields your fair balance; dishonest behavior risks losing everything.

This is not merely a social norm or implementation trick. It is encoded into Bitcoin spend conditions.

Lightning provides several major benefits over repeated on-chain payments:

- **speed:** no need to wait for block confirmations for every payment
- **cost:** fees are typically tiny compared with on-chain fees
- **scalability:** many updates compress into two on-chain transactions
- **privacy:** intermediate balance changes are not globally published
- **global reach:** users can pay peers with whom they do not share direct channels

These advantages make Lightning particularly attractive for micropayments, point-of-sale transactions, machine-to-machine transfers, and other interactive payment flows.

---

## 8. Conclusion

The Lightning Network is one of the most sophisticated constructions built on Bitcoin. It takes a minimal scripting system and, through careful protocol engineering, turns it into a high-speed payment network.

Its core ideas can be summarized as follows:

1. lock funds in a shared multisig funding output  
2. represent balance ownership with asymmetric commitment transactions  
3. revoke old states so that cheating becomes punishable  
4. use relative timelocks to create a dispute window  
5. route across the network with HTLCs  
6. rely on Bitcoin for final settlement and dispute resolution  

The result is a layered architecture in which Bitcoin remains the secure settlement base while Lightning serves as a fast payment layer.
