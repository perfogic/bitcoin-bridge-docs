# Proof-of-Stake Bitcoin Sidechains

**Matt Bell ([@mappum](https://twitter.com/mappum))** • [Nomic Hodlings, Inc.](https://twitter.com/nomicbtc)

Version 2.1 - *June 22, 2021*

## Introduction

We present a safe, practical design for a Bitcoin sidechain based on the
[Tendermint](https://tendermint.com) consensus protocol, enabling the
development of decentralized networks which coordinate to manage reserves of
Bitcoin, allowing for custom application code and smart contracts which use
Bitcoin as the native currency. We also avoid the long-range attack problem of
proof-of-stake networks by periodically timestamping the sidechain on the
Bitcoin blockchain, gaining the security of Bitcoin's proof-of-work in addition
to the instant finality of BFT consensus protocols.

Our economic design maintains security guarantees while requiring less amounts
of outside capital to be locked as collateral than other designs. Additionally,
we describe mechanisms which prevent loss of funds when up to 2/3 of the voting
power is malicious, and disincentivize malicious behavior even when 100% of the
voting power actively censors fraud proofs on the sidechain.

## Technical Overview

A sidechain based on our design exists as its own sovereign network, which we
will refer to as the **sidechain network**. The validators of this network
become the signatories of the network's reserves, each with a known
Bitcoin-compatible public key and an integer amount of "voting power",
representing their signatures' weight in the consensus process and in their
control of the reserves as governed by the Bitcoin network.

### Reserve Wallet

A **reserve** of Bitcoin is maintained in a decentralized way through use of a
special multisig contract. No individuals in the network are given custody of
the Bitcoin in reserves, but instead the collective whole cooperates to hold or
disburse funds. The validators of the sidechain network become **signatories**
of the reserve, since their signatures are required to control the funds on the
Bitcoin blockchain.

To disburse funds from the reserve, more than two-thirds of the signatory set
must sign the Bitcoin transaction (weighted by voting power). This is enforced
on the Bitcoin blockchain through the **"reserve script"**. Using the "Taproot"
family of features on Bitcoin, we can create a reserve script scheme which only
requires the on-chain size of a typical single-signature payment input and lets
us support up to 1,000 signatories.

Using a Taproot merklized abstract syntax tree-based output (MAST), we can
create a spending condition for the funds in reserve which requires either a
signature from a specified public key (the "key path"), or an input to a single
script out of a specified set of scripts (the "script path").

In our scheme, we derive the key for the key path by aggregating the keys for
the smallest set of signatories which sum to greater than 2/3 of the total
voting power (we call this subset the **"base signatory set"**). Aggregation is
done using the Musig2 N-of-N aggregated signature algorithm. We also define a
single script for the script path, the **"fallback script"** which is a large
weighted multisig script (defined below).

- **Key Path:** Aggregated public key composed of base signatory set (largest
  2/3 of signatory set by voting power)
- **Script Paths:**
  1. *Fallback script* - Weighted multisig containing all signatory public keys
     and voting power

#### Fallback Script

The fallback script is composed as follows:
```
<pubkey1> OP_CHECKSIG
OP_IF
  <voting_power1>
OP_ELSE
  0
OP_ENDIF

OP_SWAP
<pubkey2> OP_CHECKSIG
OP_IF
  <voting_power2>
  OP_ADD
OP_ENDIF

OP_SWAP
<pubkey...> OP_CHECKSIG
OP_IF
  <voting_power...>
  OP_ADD
OP_ENDIF

OP_SWAP
<pubkeyN> OP_CHECKSIG
OP_IF
  <voting_powerN>
  OP_ADD
OP_ENDIF

<two_thirds_of_total_voting_power>
OP_GREATERTHAN
```

The validators in this script are given a canonical ordering: descending by
their respective amounts of voting power, and when voting power is equal,
ordered ascending lexicographically by public key.

The key path is expected to be taken the vast majority of the time as
emperically validators on proof-of-stake networks have been shown to have
extremely high uptime. However, if only a single signatory of the base signatory
set is offline, the group will be unable to produce valid Musig2 signatures for
the aggregated public key, in which case the network will fall back to producing
standard signatures for the fallback script path.

The higher Bitcoin network fee cost of the fallback script path can possibly be
rectified by forcing the offline signatory (or signatories) of the base set to
pay for the increased fee out of their stake, also creating a disincentive for
preventing the key path spend.

#### Limits

An input which spends the key path will use about 30 virtual bytes (the same as
a typical payment transaction) for any number of signatories. A spend of the
fallback script path will have about 3,000 virtual bytes for 100 signatories.

The Bitcoin consensus rules remove script size limits for Taproot MAST scripts,
so teh fallback script can technically be any size. However, there is an added
consensus rule which limits the size of the Taproot spend's initial stack to
1,000 elements which means a spend of the fallback script can not include more
than 1,000 signatures - limiting us to exactly 1,000 signatories. If there is a
1-to-1 mapping of validators of the sidechain network to signatories, then this
limit is more than enough as Tendermint-based networks have on the order of 100
validators. The increased signatory set size limit could also possibly be used
to increase security by allowing for other staked nodes to participate as
signatories even though they are not Tendermint validators.

#### Signing Process

For both key-path spends and fallback-path spends, producing a signature for the
reserve script will involve a process where signatories submit their individual
**"signature shares"** to the sidechain where the state machine will collect
them on the state so valid signatures can be assembled by relayers. For the
key-path, the signature is complete once all signatories in the base set have
submitted a valid signature share. For the fallback-path, the signature is
complete once at least two-thirds of the voting power is represented in valid
signature shares.

For the Musig2 aggregated signature used in the Taproot script, a first round
must happen where signatories commit to nonces used in the signing process, but
this can happen separately before there is a message to be signed.

After a timeout, if the base signatory set has not completed its aggregated
signature, the entire signatory set can work to produce a spend of the fallback
script. While this could also happen in parallel to reduce latency in the
fallback case, it may possibly allow malicious or faulty relayers to broadcast
valid fallback-path Bitcoin transactions when their key-path alternative was
also signed, causing the transaction to incur more Bitcoin network fees than
necessary.

### Deposits

To move funds into the reserve pool, depositors send a Bitcoin transaction which
pays to the reserve script (derived based on the sidechain network's known
signatory set), along with a commitment to the desired destination address in
the sidechain network ledger.

There are multiple possible deposit schemes:

#### Simple OP_RETURN Commitment Scheme

This kind of deposit transaction must have exactly two outputs:

1. A pay-to-taproot that pays to the hash of the reserve script. All the coins
   to be deposited into the reserve are paid into this output.
2. A script which commits to a destination address which will receive the pegged
   Bitcoin on the sidechain network ledger (`OP_RETURN <sidechain address
   bytes>`). This output has an amount of zero.

This scheme is simple for relayer nodes to detect since they can simply filter
Bitcoin transactions by checking for outputs which pay to the current reserve
script.

However, this scheme faces an issue: currently, no Bitcoin wallet or address
scheme will construct transactions with this format. In practice, deposits using
this scheme will often be made up of two separate transactions: 1) a standard
payment transaction which sends from the user's conventional Bitcoin wallet to a
key controlled by their sidechain wallet, and 2) a specialized deposit
transaction as described above, constructed by the sidechain wallet.

If sidechain transactions became a common occurence in the future, wallets could
possibly adopt a new address scheme which is able to instruct the wallet to
create a valid sidechain deposit transaction which pays to a given signatory set
and commits to a given address.

#### Script Path Commitment Scheme

In this scheme the deposit transaction doesn't need a special format. The
reserve script is modified to contain an extra script in its script path tree,
so a wallet can send a valid deposit transaction with a standard BIP173 address.

The commitment would be a standalone script path which can never be spent:
`OP_RETURN <sidechain address bytes>`.

The Taproot construction for the deposit output would then look like the
following:

- **Key Path:** Aggregated public key composed of base signatory set (largest
  2/3 of signatory set by voting power)
- **Script Paths:**
  1. *Fallback script* - Weighted multisig containing all signatory public keys
     and voting power
  2. *Deposit address commitment* - An `OP_RETURN` to indicate where the pegged
     BTC should be credited on the sidechain

This scheme doesn't have the issues of the formerly mentioned scheme, however
without any advance knowlege, it is impossible for relayer nodes to detect that
a transaction is a deposit of this type since the reserve output contains a
unique hash due to its unique commitment.

To detect the outputs, depositors will need to broadcast their address to
relayers off-chain, and relayers will need to store the address for some
reasonable period of time (e.g. hours or days). Then, when scanning for
deposits, relayers will need to match every Taproot output they see in Bitcoin
transactions against all currently valid signatory sets and possible deposit
commitment addresses. Even though they may need to match with a very high number
of possible deposit outputs, the operation can be made cheap with hash-based
data structures (e.g. a hashmap or Bloom Filter).

This scheme is ideal from a user-experience perspective, since a user can
generate a deposit address, send to it from their standard Bitcoin wallet, then
once the transaction is relayed the funds should show up in their sidechain
account - very similar to the process of depositing into a traditional
centralized platform. Wallets can also transparently broadcast deposit addresses
to relayers in the background without this becoming an explicit extra step.

#### Timelocked Deposit Reclamation

It may be possible for a deposit to be made to an outdated reserve script -
either by the deposit transaction taking a long time to confirm due to
underpayment of fees or unexpectedly high load on the Bitcoin network, or
possibly by a software issue or user error. Since the sidechain does not honor
deposits to outdated reserve scripts (since the signatories are not necessarily
available anymore, or may have unbonded their stake), this would by default
result in loss of funds.

To solve this, we modify the reserve script slightly to make it possible for the
depositor to reclaim the funds after a given time or block height. This is
implemented in the Taproot reserve script construction by adding an additional
script path, containing the following script:

```
<locktime> OP_CHECKLOCKTIMEVERIFY OP_DROP
<depositor_pubkey> OP_CHECKSIGVERIFY
```

Assuming we are using the script path commitment scheme as mentioned above, the
Taproot construction for a deposit looks like the following:

- **Key Path:** Aggregated public key composed of base signatory set (largest
  2/3 of signatory set by voting power)
- **Script Paths:**
  1. *Fallback script* - Weighted multisig containing all signatory public keys
     and voting power
  2. *Deposit address commitment* - An `OP_RETURN` to indicate where the pegged
     BTC should be credited on the sidechain
  3. *Deposit reclamation script* - Must be signed by depositor, only valid
     after timelock has passed

The `timelock` field should be set to a time or block height in the future
significantly past the time it would take for the deposit transaction to be
confirmed on the Bitcoin network, relayed to the sidechain network, collected
into a checkpoint, and have the checkpoint be confirmed on the Bitcoin network.
For example, this could be on the order of 1 day or 144 blocks in the future.
The choice of this field can be left up to the depositor, but a minimum value
should be enforced by the network to prevent a denial-of-service attack where
the depositor reclaims the deposit just before the sidechain signatories are
about to broadcast their spend of it (however, this attack is not an issue if
the network is able to detect reclaimed deposits and abort spending them).

Note that the funds will still be spendable by the outdated signatory set after
the timelock, so the depositor isn't guaranteed to recover their funds and
should broadcast a transaction reclaiming the funds as soon as the timelock
passes. However, if the stake unbonding period is significantly longer than the
time before the reclamation timelock, signatories can be slashed for signing a
spend of a deposit which has matured past its timelock. Depositors may also
pre-sign a reclamation transaction and send this to "watchtower" nodes who may
broadcast it on their behalf for extra assurance that the reclamation happens in
a timely fashion.

#### On-Chain Deposit Process

Once a relayer node detects a valid Bitcoin transaction which has outputs
matching the above format, it will broadcast a **deposit proof** to the
sidechain network, which contains:

- the bytes of the complete deposit transaction data
- the address bytes committed to in the deposit transaction *(when using the
  script path commitment scheme)*
- the hash of the Bitcoin block which contained the deposit transaction
- the Merkle branch proving the transaction was included in the Bitcoin block

After the sidechain network receives a valid deposit proof, it will then mint
pegged tokens on its ledger, paid out to the destination address committed to by
the depositor. These pegged tokens represent claims on the Bitcoin in reserves,
which can be transferred, used in smart contracts, or burnt to trigger a
withdrawal from the reserves paid to a given destination on the Bitcoin
blockchain.

#### Deposit Finality

Similar to other deposit-accepting systems, such as traditional exchanges and
merchants, the sidechain accepts the risk of considering a deposit final
(therefore granting pegged tokens) but then having the depositor reverse the
transaction through e.g. a reorg of the Bitcoin blockchain. To minimize
vulnerability to these kinds of attacks, the system should not consider a
deposit final until it has been confirmed sufficiently deep on the Bitcoin
blockchain.

Typical deposit-acceptors simply wait for a transaction to be N blocks deep,
however this can still leave the system vulnerable to attacks, especially when
transfers are allowed (in contrast to merchant platforms, where a double-spend
has limited damage since an administrator can simply cancel the attacker's
pending orders and remove the balance from their account).

A safe heuristic would be to wait for at least the pending quantity of Bitcoin
to be mined for a deposit to progress from "pending" to "final". For instance,
if 8 BTC is deposited, and 6.25 BTC is mined per block on mainnet, then waiting
for 2 confirmations is a safe depth since a miner would likely consume more
costs in a chain reorg than in gains they made from the fraudulent deposit.

The finality heuristic should also not make the assumption that every depositor
is a separate party - if deposit confirmations simply depended on the amount of
BTC in the UTXO then a Sybil attack would be possible where the deposit is split
into many smaller-sized transactions which are all double-spent together. A
conservative solution would be to keep a network-wide counter of the total BTC
quantity currently being deposited, then scaling confirmation block counts for
all deposits regardless of size based on this total, confirming the individual
deposits in a FIFO ordering.

Since these very conservative confirmation heuristics create a degraded user
experience compared to traditional centralized platforms which often only wait
for 1 or 2 block confirmations, we can create a market for speculation on
transaction confirmations so that users can have instant access to their funds
minus some premium, and nodes with advanced knowlege of the Bitcoin network can
earn revenue by modelling the risk of double-spends. We expand on this more in
the **Deposit Confirmation Derivatives** section in the appendix of this
document.

#### Deposits to Outdated Signatory Sets

The signatory set may have changed after a deposit was sent but before
confirmation, so deposits to slightly-outdated signatory sets must still be
considered valid. If the unbonding period is sufficiently long (on the order of
days), we can ensure deposits to stale signatory sets are safe and process them
up to some maximum age which is significantly less than the unbonding period.

Deposits which are not processed can be reclaimed based on the timelocked
reclamation process, and can still be considered safe until the signatory set is
older than the unbonding period. If the unbonding period is significantly
longer than the deposit reclamation period then depositors should be able to
reclaim without risk.

### Checkpoints

Periodically, the network will make transactions on the Bitcoin blockchain which
spend from the reserve wallet. These transactions are called **checkpoints**,
and serve the purpose of (1) collecting deposits, (2) updating the reserve
script to reflect the latest signatory set, (3) disbursing pending withdrawals,
(4) providing a way for light clients to verify the state of the sidechain
network secured by the Bitcoin network's proof-of-work, and (5) invalidating the
previous "emergency disbursal" transaction (mentioned later in this document).

![](./diagram.svg)

Each checkpoint is made up of 3 connected Bitcoin transactions, the **deposit
collection** transaction, the **checkpoint** transaction, and the **disbursal**
transaction.

#### Deposit Collection Transaction

A deposit collection transaction spends all sufficiently confirmed unspent
deposit outputs and joins them into a single output. It has a variable number of
inputs, depending on the number of pending deposits. It always has exactly one
output, paying all funds minus some fee to the reserve script. If no deposits
have been made, no deposit collection transaction will be made.

#### Checkpoint Transaction

A checkpoint transaction spends from the latest deposit collection output, and
the output of the previous checkpoint transaction. It will have the following
structure:

**Inputs:**
1. The reserve output of the previous checkpoint transaction.
2. *(If there have been deposits)* The deposit collection transaction output.

**Outputs:**
1. The **reserve output**, equal to the amount of Bitcoin which are to be held
   in reserve. Paid to the updated reserve script based on the most recent
   signatory set.
2. *(If there are pending withdrawals)* The **disbursal output**, equal to the
   total amount of Bitcoin to be disbursed. Paid to the updated reserve script
   based on the most recent signatory set.

#### Disbursal Transaction

Disbursal transactions spend the second output of the most recent checkpoint
transaction, and pay to various outputs to settle any pending withdrawals. Each
output pays its respective amount to the script specified in its withdrawal
request. If no withdrawals are pending, this transaction is not created.

#### "Follow-the-Money" Proof-of-Stake Verification

A known issue of proof-of-stake consensus is the so-called *long-range attack*,
where a client verifying the blockchain cannot safely sync if their most recent
knowledge about the network is out of date (e.g. the validator set they last
knew about may now have *nothing at stake* and would be able to trick the client
into accepting an alternate ledger, while suffering no risk of having their
stake taken away).

This kind of issue can be solved out-of-band from the proof-of-stake network,
e.g. by receiving new knowledge about the network from a trusted third party.
However, our Bitcoin checkpointing mechanism allows clients to prevent
long-range attacks by utilizing the proof-of-work security of the Bitcoin
blockchain.

To securely sync through history to get the latest state of the proof-of-stake
sidechain network, a client will first SPV-verify the headers of the Bitcoin
blockchain, ensuring they are on the highest-work chain. After this, the client
only needs to possess each checkpoint transaction and its Merkle branch proving
its membership in the containing Bitcoin block. The client can securely verify
that a checkpoint transaction is the successor of another by ensuring it spends
the *reserve output* of the previous checkpoint transaction. By following this
chain of checkpoint transactions, the client can ensure that a signatory set is
the correct one by deriving its reserve script and comparing to the one in the
latest reserve UTXO.

While the three transaction types described (**deposit collection**,
**checkpoint**, and **disbursal**) could be combined into one for simplicity and
space savings, we separate these for the purpose of reducing the amount of data
for a light client to follow the chain of checkpoints. If all deposits and
withdrawals were also contained in the checkpoints, the light client would need
to download all this data just to sync through history (Bitcoin transaction and
not Merklized - all inputs and outputs are required to calculate a transaction's
hash). On a large-scale sidechain network, this data could be a significant size
to a light-client.

#### Interval

To keep the signatory set as reflected on the Bitcoin blockchain as close as
possible to its current state on the sidechain, a checkpoint should be created
every time the signatory set changes by a certain threshold, or on a certain
time interval. A faster checkpoint interval has higher Bitcoin fee cost, but
also decreases the processing time for deposits and withdrawals. For a
large-scale network, we expect the network's transaction fee cost to be
negligible compared to its reserves, so it's feasible for the network to aim to
produce the ideal of one checkpoint per Bitcoin block.

### Relaying

Whenever a new Bitcoin block is mined, or a deposit transaction is broadcast to
the Bitcoin network, the data will need to be carried to the sidechain network.
Conversely, when a transaction is signed by the signatory set in the
checkpointing process, it will need to be carried to the Bitcoin network. This
job is done by **relayer** nodes, which can be any node with knowledge of both
networks running software to broadcast the relayed data.

Note that no trust is placed in the relayer nodes and the system operates
correctly as long as at least a single relayer node is active. In practice a
significant amount of sidechain full nodes will likely opt to run relayers.

**Relayed from Bitcoin to sidechain:**
- *Bitcoin block mined* - header is relayed to sidechain
- *Deposit transaction is confirmed* - transaction and Merkle proof are relayed
  to sidechain

**Relayed from sidechain to Bitcoin:**
- *Checkpoint signed by signatory set* - assembled transaction is broadcasted to
  Bitcoin network

### Security Features

Since sidechain networks may one day hold large reserves of Bitcoin, security
needs to be carefully considered. In contrast with many blockchain projects, a
security incident resulting in the loss of funds can not be reverted through
consensus of the sidechain network, and appealing to Bitcoin governance for a
bailout is considered impossible.

Sidechains backed by decentralized custody like the one described in this
document inherently suffer from risk of failure, where signatories take the
money from the reserve (by collusion among at least 2/3 of the voting power, an
endemic software vulnerability, or some other reason). While this mechanism
offers many improvements over a centralized party who may act arbitrarily, it
makes sense to make efforts toward reducing all possible risk of sidechain
failure.

We reduce risk of failure of the sidechain through a staked token which acts as
collateral for signatories, an emergency disbursal process which protects
against liveness failures, and a stake-locking mechanism which ensures that
signatories are economically disincentivized from stealing from the reserves
even when they are able to censor fraud proofs on the sidechain.

#### Staking Token

Native to the sidechain network is a token which may be locked or "staked" to
gain voting power as a validator in the network's consensus process in addition
to voting power as a signatory in the network's decentralized custody of its
Bitcoin reserves. As with all modern proof-of-stake networks, the staked tokens
are "slashed" when fraud is proven, for instance in a "double-signing" attack.

In our case, proof of an unexpected signature for a Bitcoin transaction which
spends from the reserves is also a slashable offense since signatories should
only ever sign transactions which spend the reserves for the network's
checkpointing process. This can be proven to the network by submitting a **fraud
proof** transaction which contains the (partially) signed Bitcoin transaction -
the network state machine should be able to parse the Bitcoin transaction,
verify that it is an unexpected spend of the reserves, and verify that one or
more signatories signed it, and then slash those signatories' stake accordingly.

Many designs exist to define how the staking token comes into circulation. In
order to maintain a fair and decentralized distribution, for instance, tokens
could be minted based on a submitted proof of burned BTC, or to Bitcoin miners
as a secondary block reward.

#### Reserve Demurrage

In common with other decentralized Bitcoin custody projects backed by collateral
such as tBTC and XCLAIM, a simple heuristic exists where the signatories must be
overcollateralized relative to the quantity of Bitcoin reserves they are
managing. We'll call the ratio between the signatories' total staked collateral
and the reserves the **"collateralization ratio"**. If the collateralization
ratio was less than 1.5, a rational 2/3 subset of signatories would simply steal the reserves (they
would benefit even if they lose all of their collateral). However, this
requirement is typically very capital inefficient - for a network to safely hold
$1B in its reserves, signatories would need to invest more than $1.5B in other
outside capital as collateral. This requirement is also assumed to have
prevented adoption of these decentralized custody projects in favor of
centralized trusted custodian-based projects such as Wrapped Bitcoin.

To reduce the amount of outside capital required to satisfy this security
heuristic, our native staking token earns recurring revenue by charging
demurrage (negative interest) from all accounts holding claims to BTC on the
sidechain. Based on this revenue, the value of the staking token can be valued
relative to the size of the reserves based on the discounted cash flow - as more
Bitcoin is deposited into the system, the value of the stake should increase
proportionally.

The demurrage rate paid by BTC claim-holders is called the **reserve rate** and
is continuously paid to holders of staked staking tokens.

The reserve rate and collateralization ratio can be set by various different
schemes, for example:
- **Collateralization-Adjusted Rate:** If we assume we have some oracle that
  gives us the price between the staking token and Bitcoin (e.g. a TWAP
  automated market maker hosted on the sidechain), we can calculate the
  collaterization ratio. We can choose some **target collateralization ratio**,
  then if the staked value is too low the reserve rate can be increased, and if
  the staked value is higher than necessary the reserve rate can be decreased
  (e.g. with a simple control algorithm such as a PID controller).
- **Capped Collateralization:** Another scheme would require all Bitcoin
  claim-holders to choose a maximum reserve rate. Then, if the collateralization
  ratio falls below its target, BTC-claims would be removed from the network
  through a forced disbursal, with priority for keeping the claims with higher
  maximum reserve rates. When the collateralization ratio is in its target
  range, the effective reserve rate paid can be given by the lowest chosen
  maximum out of the BTC-claim accounts on the chain.

If these schemes are based simply on the price of the stake, then they are not
factoring in liquidity - the stake may be less valuable than the reserves since
it cannot be effectively liquidated for a high enough value. Because of this,
the collateralization ratio should possibly be higher than 1.5.

It should be noted that any complementary added source of value for the staking
token, such as its potential revenue for participation in the consensus process,
should decrease the effective reserve rate paid by claim-holders. The reserve
rate can be thought of as a fee paid to maintain security (similar to the
dilutive block reward in Bitcoin), where the staked signatories of the chain are
the ones providing the security and taking on operational risk with their
captial.

#### Emergency Disbursal Process

In the case of an extended liveness failure, all deposited funds would be frozen
in place on the Bitcoin blockchain with no recourse other than manually
resolving the situation with the signatories.

To protect against this case, as part of the checkpointing process signatories
also sign an **"emergency disbursal"** transaction which spends the entire
reserve and pays out each individual claim-holder on the Bitcoin blockchain.
Signatories publish these signatures to the network at the time of the
checkpoint so that relayers may assemble them.

These transactions are timelocked at some future date, e.g. 2 weeks past the
checkpoint, so that the emergency disbursal only happens if a checkpoint has not
been created for an extended period of time. Since Bitcoin signatures commit to
the inputs they spend, signed emergency disbursals will automatically be
invalidated as soon as a newer checkpoint happens since it will spend the same
reserve UTXO and become confirmed before the emergency disbursal unlocks.

In the case an emergency disbursal actually happens, the remaining stake becomes
valueless and the signatories have all effectively been entirely slashed - so it
is expected that the signatories will do everything in their power to prevent
extended periods of liveness failure.

Bitcoin has default policies to limit transactions to 100K virtual bytes, so for
a large-scale sidechain this would be structured as a tree of multiple
transactions in order to scale to contain 1 output per account. Assuming there
are 100,000 accounts on the sidechain, and each of these accounts receive a
pay-to-pubkey-hash UTXO (34 bytes each), this would require over 3.4M vbytes of
transaction data (a small amount of additional space is required for the base
transaction fields and the inputs). This structure could then be made up of
about 36 transactions, 1 intermediate transaction which directly spends the
reserve output and has 35 outputs, and 35 transactions which spend outputs from
the intermediate transaction and pay out to many P2PKH outputs (one for each
account).

#### Stake-Locking

On-chain slashing mechanisms rely on honest nodes submitting fraud proofs to the
network. However, in a scenario where more than 2/3 of the voting power is
executing an attack (e.g. stealing from the reserves), the attackers also
control the consensus for the sidechain network and are able to censor fraud
proofs to avoid losing any of their stake. To prevent this, we introduce a
simple rule where signatories who request to unbond do not receive their stake
until after proof of a checkpoint committed to the Bitcoin blockchain which
reflects the unbonding in its updated signatory set.

This means that even if the attackers control 100% of the voting power and have
the power to censor fraud proofs against themselves, they are unable to receive
their stake without actually removing their own control from the reserves on the
Bitcoin blockchain. In a scenario where attackers successfully commit an
unexpected spend from the reserves, all stake would be forfeited - it would be
impossible to unlock as now a valid checkpoint is impossible to produce.

## Appendix

### Deposit Confirmation Derivatives

Once a deposit transaction on the Bitcoin blockchain is considered final and the
depositor has been paid out in BTC-claim tokens, the process cannot be reversed
on the sidechain even if the deposit is reversed on the Bitcoin blockchain since
the tokens may have been transferred to other parties.

This means the sidechain should only grant BTC-claim tokens when it is entirely
certain the Bitcoin transaction cannot be reversed, which is why we use
conservative heuristics for considering a deposit final. Because of this,
depositors are required to wait for longer confirmation periods than they may be
used to on centralized platforms.

To give depositors the ability to access their funds instantly without requiring
the network to take on risk of double-spends, we can create a market where
traders may speculate on the likelihood of a deposit being confirmed.

To implement this market, we can allow for relaying of deposits which have not
yet been confirmed based on the sidechain's finality conditions - including
deposits which have not yet been mined in a Bitcoin block at all. Once one of
these non-final deposits are relayed, a new token specific to this deposit UTXO
is minted to the depsitor, with an equivalent quantity to what will be granted
in BTC-claim tokens once the deposit is confirmed. Upon confirmation of the
deposit, these tokens are convertible at a 1-to-1 ratio for newly-minted
BTC-claim tokens from the network.

In order to access the funds without waiting for the confirmation, a depositor
may sell these unconfirmed-deposit tokens to traders who have beliefs about the
deposit's likelihood to eventually be confirmed (through data such as the
deposit's prevalence in BTC miners' mempools, whether the deposit opts into
replace-by-fee, network hashrate conditions indicating likelihood of a reorg,
knowlege about the depositor, etc.). Tokens will trade at a market price less
than 1, with the difference depending on factors such as perceived risk, the
time-value of money until the deposit is confirmed, and liquidity in the
unconfirmed deposit market. Depositors are essentially paying a premium for
access to instant funds and traders are earning the premium for taking on the
double-spend risk.

### Liveness Failure Swaps

We can separate BTC-claim tokens into two components, 1) withdrawal-claims which
grant the holder the right to withdraw for an equal amount of BTC on the Bitcoin
main chain via the disbursal process, and 2) emergency-disbursal-claims which
converts to main chain BTC for the holder in the event of a liveness failure via
the emergency disbursal. If there exists a market between the two, traders can
speculate on the likelihood of an emergency disbursal happening. This market can
serve as an important indicator to the rest of the network by measuring the
risks of a catastrophic failure.

However, it should be noted that this market will not factor in the risk of
theft of the reserves by the signatories since this will spend the reserve
output and invalidate the emergency disbursal.
