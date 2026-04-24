# The Sovereign Boardroom
## How It Works — Member Guide

*Version 1.0 — Draft*

---

## What you have

**The SBT** (Sovereign Boardroom Terminal) is a small handheld device with a touchscreen. It has no software installed. It has no memory between sessions. It is a screen and a processor — nothing more. You can buy one from any electronics supplier or from our shop.

**The SBKey** is your key to the Boardroom. It is a small metal fob on your keyring. Inside it, permanently sealed, is a microSD card carrying your encrypted credentials. Without your SBKey, you cannot participate in any Boardroom decision. Without your PIN, your SBKey is a blank piece of hardware.

---

## What the Sovereign Boardroom is

A shared treasury controlled jointly by a defined group of members. No single member can move funds alone. Transactions require a minimum number of members to agree and sign — a number set by the group at the founding ceremony and fixed permanently thereafter.

There is no company in the middle. No account to log into. No app that can be updated without your knowledge. No third party who can freeze, delay, or monitor your transactions. The Boardroom exists on the Bitcoin network and is governed entirely by the keys your members hold.

---

## The three phases

### Phase 1 — The founding ceremony

The group meets — physically or across verified devices — to establish the Boardroom. This happens once. It cannot be undone or modified without starting again from the beginning.

During the ceremony:

- The group agrees on the signing threshold. If there are seven members, the group might require four signatures to authorise any transaction. This number is permanent.
- Each member's SBKey is generated independently. No member sees another member's key. No key ever leaves its SBKey.
- A shared treasury address is created from the combined public keys of all members. Funds sent to this address can only be moved with the required number of signatures.
- Each member leaves with their SBKey, their PIN, and the treasury address.

After the ceremony, no single person — including whoever organised it — has any privileged access. The group governs by consensus. The software has no concept of administrator.

> **Technical layer:** Each SBKey contains an encrypted BIP-32 HD wallet seed. The treasury is a Bitcoin multisig address constructed from the public keys of all members using standard P2WSH or P2TR encoding. The signing threshold is encoded in the redeem script and is verifiable on-chain by anyone.

---

### Phase 2 — Proposing a transaction

Any member can propose a transaction at any time. No permission is required to propose. A proposal is a request — it becomes a transaction only when enough members have signed it.

To propose:

1. Insert your SBKey into the SBT
2. Enter your PIN
3. Select *New Proposal*
4. Enter the destination address and amount
5. The SBT constructs a partially signed transaction and broadcasts the proposal to the group

The proposal travels between members as a standard Bitcoin PSBT — a Partially Signed Bitcoin Transaction. This is a file, not a secret. It contains the transaction details and accumulates signatures as members approve it. It carries no private key information.

Members receive the proposal and review it before signing. Nothing moves until the threshold is reached.

> **Technical layer:** Proposals are distributed via Nostr — a decentralised, censorship-resistant messaging protocol. The PSBT format is defined in BIP-174. The SBT signs only the transaction hash, never exposing the private key. What travels over the network is a cryptographic signature — mathematical proof that the key was used.

---

### Phase 3 — Signing and broadcast

When a member decides to approve a proposal:

1. Insert SBKey into the SBT
2. Enter PIN
3. Review the transaction — destination, amount, fee
4. Confirm
5. Remove SBKey

The SBT adds the member's signature to the PSBT and passes it along. When the final required signature is added, the SBT broadcasts the completed transaction directly to the Bitcoin network. No intermediary. No delay. No approval required from any third party.

Once broadcast, the transaction is final. Bitcoin transactions are irreversible by design. The Sovereign Boardroom does not add a cancellation window or a reversal mechanism. Review carefully before confirming.

> **Technical layer:** Broadcast is made directly to the Bitcoin peer-to-peer network via a node connection configured during setup. Alternatively, the completed transaction hex can be broadcast manually via any Bitcoin node or public broadcast service. The SBT performs a mandatory RAM wipe after the SBKey is removed — no session data persists on the device.

---

## What happens if something is lost

**If you lose your SBT:** Buy another one. Load the Sovereign Boardroom firmware from our verified GitHub repository. Your SBKey works in any SBT. The terminal holds nothing — everything is in the key.

**If you lose your SBKey:** This is serious. Your key cannot be reconstructed without the backup. Inform the other members immediately. Your threshold of remaining members can still operate the treasury — they do not need your signature to transact, provided enough of them remain. However, your key is gone. The group should discuss whether to migrate the treasury to a new Boardroom with a new set of keys.

**If you forget your PIN:** The SBKey cannot be unlocked without the correct PIN. There is no recovery mechanism. This is by design — a recoverable PIN is a vulnerability. The PIN should be memorised and never written down in connection with the SBKey.

**If the backup SBKey is lost:** Treat this as urgent. The backup is a second sealed fob held in a separate location. If both the primary and backup SBKeys are lost, that member's key is permanently gone. Act before this happens.

---

## What the Sovereign Boardroom cannot do

It cannot reverse a transaction. It cannot freeze funds. It cannot lock a member out. It cannot be updated without your knowledge — the firmware is open source and the version on your SBKey is fixed at the point of writing.

It does not know who you are. It does not record your transactions. It does not communicate with any server operated by Sovereign Boardroom. Once your SBKey is written and sealed, your relationship with us is complete.

---

## What it looks like in practice

A board of five members controls a shared operational fund. They set a 3-of-5 threshold at founding.

A payment needs to go out. One member proposes it on their SBT — takes about ninety seconds. The proposal appears on the other members' devices. Two more members review and sign — each takes about sixty seconds. The transaction broadcasts automatically when the third signature is added.

Three members in three different cities. No meeting required. No bank to call. No compliance officer to approve. The transaction is on-chain within minutes of the third signature.

No member knows the others' PINs. No member has seen the others' SBKeys. No member could have moved the funds alone.

---

## A note on responsibility

The Sovereign Boardroom is a tool for groups who take shared responsibility seriously. It removes third parties from your treasury — and with them, the safety nets those third parties provide. There is no customer support line that can retrieve lost funds. There is no dispute resolution process. There is no undo.

This is the point. Sovereignty means the decisions are yours, the keys are yours, and the consequences are yours.

Treat your SBKey accordingly.

---

*The Sovereign Boardroom — sovereign-boardroom.com*
*Firmware: github.com/sovereign-boardroom — audited, open source*
