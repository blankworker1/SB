**WESATOSHIS LABS**

**Sovereign Boardroom**

Firmware Module Specification

*3-of-5 Geographically Distributed Multi-Signature Treasury*

Version: 0.1 Draft

Classification: Confidential --- Internal Technical

Date: April 2026

Target hardware: Wesatoshis Card (Wi-Fi + SPV node variant)

**1. Purpose and scope**

This document specifies the firmware module --- codenamed Sovereign Boardroom --- to be developed for the Wesatoshis Card. The module extends the card\'s existing payment capabilities to support a geographically distributed 3-of-5 multi-signature Bitcoin treasury, operable entirely from hardware cards with no companion laptop or coordinator software required.

The confirmed aim of this specification is:

> *To construct a multi-sig platform that allows 3-of-5 devices to enable five card holders in different geographic locations to build, coordinate, and sign Bitcoin transactions using only their Wesatoshis Cards and a Nostr-compatible messaging application on their mobile phones.*

This module is scoped to Bitcoin mainnet transactions using P2WSH (native SegWit) multi-signature outputs. Rootstock (RSK) and Lightning Network extensions are out of scope for version 1.0.

**2. Context and design philosophy**

Existing multi-signature treasury solutions require one of two compromises: either a general-purpose computer running coordinator software (Sparrow, Specter Desktop), or trust in a custodial third party (Unchained, Casa). Neither option is acceptable for a community treasury that prioritises censorship resistance, physical mobility, and air-gap-equivalent security.

The Wesatoshis Card\'s combination of Wi-Fi connectivity, an embedded SPV node, and existing Nostr integration creates a unique hardware basis for a card-native coordinator. No other consumer hardware wallet currently offers this capability.

**2.1 Target user profile**

The target user is a competent Bitcoin self-custodian who:

- Understands self-custody and holds a single-signature cold wallet

- Has heard of multi-signature wallets but has no practical experience creating or signing a multi-sig transaction

- Uses a Lightning wallet for day-to-day transactions

- Communicates with their board via Nostr or a Nostr-compatible messenger

The module must make the complexity of multi-signature coordination completely invisible. Technical terms such as XPUB, PSBT, and descriptor must not appear in the user interface.

**2.2 Non-goals for version 1.0**

- Speed optimisation --- correctness and security take priority over transaction throughput

- Lightning Network channel management

- Rootstock / RSK smart contract interaction

- Mobile companion app --- Nostr messaging apps (Damus, Amethyst, Primal) serve this role

- More than one active treasury per card --- a single treasury per card is sufficient for v1

**3. System architecture**

The Sovereign Boardroom module sits as a distinct firmware layer above the existing Wesatoshis payment stack. It introduces three new subsystems: the Treasury Manager, the PSBT Engine, and the Nostr Relay Client.

**3.1 Component overview**

|                        |                                                                                                                          |
|------------------------|--------------------------------------------------------------------------------------------------------------------------|
| **Component**          | **Responsibility**                                                                                                       |
| Treasury Manager       | Stores the treasury descriptor, member XPUBs, derivation paths, and shared address. Handles enrolment and re-enrolment.  |
| PSBT Engine            | Creates, parses, partially signs, merges, and validates Partially Signed Bitcoin Transactions per BIP-174.               |
| SPV Node (existing)    | Provides block header sync, Merkle proof verification, UTXO existence checks, and transaction broadcast.                 |
| Nostr Relay Client     | Publishes and subscribes to encrypted Nostr events carrying PSBT payloads. Implements NIP-44 encryption.                 |
| Key Manager (existing) | Holds the card\'s BIP-32 HD seed in the secure element. Provides signing operations. Never exports private key material. |
| UI Layer               | Card screen and input handling. Renders treasury-specific flows: Setup, Pending, Sign, Broadcast.                        |

**3.2 Communication model**

All inter-card coordination is asynchronous and relay-mediated. Cards do not communicate directly with each other. The Nostr relay acts as an encrypted shared inbox.

- Each treasury is associated with a dedicated Nostr keypair, derived deterministically from the treasury descriptor hash.

- PSBT payloads are published as NIP-44 encrypted Nostr events, keyed to the treasury keypair.

- Each card maintains a WebSocket subscription to the treasury event stream. No polling is required.

- The relay operator cannot read PSBT contents --- encryption is end-to-end between cards.

- Two relay endpoints are required at minimum. If the primary is unreachable, the card falls back automatically.

**3.3 Bitcoin standards compliance**

|              |                                                                       |
|--------------|-----------------------------------------------------------------------|
| **Standard** | **Usage**                                                             |
| BIP-32       | HD key derivation for member XPUBs                                    |
| BIP-48       | Derivation path for multi-signature wallets (purpose field = 48\')    |
| BIP-67       | Lexicographic key sorting for deterministic P2WSH script construction |
| BIP-174      | PSBT format for unsigned and partially-signed transactions            |
| BIP-157/158  | Compact block filters for SPV privacy-preserving UTXO verification    |
| BIP-380      | Output descriptor format for treasury configuration sharing           |
| P2WSH        | Native SegWit multi-sig output script type                            |

**4. Core workflows**

**4.1 Phase 1 --- Treasury setup (one-time)**

This phase is completed once. It is repeated only if a board member is replaced.

**Step 1 --- Generate and share invite**

- Board member opens the Treasury section on their card.

- Card derives the member\'s XPUB at the BIP-48 path and encodes it alongside the derivation path as a compact string (the Treasury Invite).

- The invite string is displayed on screen and can be copied into any Nostr messenger for distribution to the other four members.

- No private key material is included in or derivable from the invite string.

**Step 2 --- Assemble the descriptor**

- One designated member (the Setup Coordinator, typically the CEO) collects all five invite strings.

- Their card parses the five XPUBs, sorts keys per BIP-67, constructs the wsh(sortedmulti(3,\...)) output descriptor, and derives the treasury P2WSH address.

- The descriptor string is shared back to the group via Nostr.

**Step 3 --- Import and verify**

- Each of the remaining four cards imports the descriptor string.

- Every card independently derives the treasury address from the descriptor and displays it on screen.

- All five members verbally or via Nostr confirm they see the same address. This is the Setup Ceremony.

- If any card displays a different address, setup fails loudly and must restart. This indicates a tampered descriptor.

> *The Setup Ceremony is the single most important security moment in the entire workflow. The address displayed on all five cards must match exactly before any funds are deposited.*

**4.2 Phase 2 --- Transaction proposal**

Any card holder may propose a transaction. In practice this will typically be the CEO or treasurer.

- The proposer opens the Treasury app and selects Send.

- They enter the destination address and amount using the card\'s input interface.

- The card constructs a PSBT: it selects UTXOs from the treasury address, sets the destination output, calculates the change output back to the treasury address, and applies a fee rate based on current mempool data from the SPV node.

- The card adds the first partial signature (sig \#1 of 3 required).

- The card performs SPV verification: it confirms the input UTXOs exist and are unspent using BIP-157/158 compact filters.

- The PSBT is published as a NIP-44 encrypted event to the treasury Nostr keypair.

- The card displays a confirmation screen showing the transaction ID, destination address (full, untruncated), amount, change amount, and fee.

> *The proposer should communicate the transaction details to the board via a separate human channel (Nostr group message, voice call) before other members sign. The card cannot enforce this policy but the operational procedure should require it.*

**4.3 Phase 3 --- Signing**

Any board member whose card is enrolled in the treasury can sign. The first three to sign complete the threshold.

- The signer\'s card receives the pending PSBT via the Nostr WebSocket subscription and displays a Pending badge.

- The signer opens the Pending screen. The card downloads the latest PSBT version from the relay.

- The card performs independent SPV verification of the PSBT: it checks that the input UTXOs are unspent and that the output address is a valid Bitcoin address.

- The card displays the full transaction details: destination address (full, untruncated), amount, change address, change amount, fee, and current signature count.

- The signer explicitly confirms by pressing the sign button. No implicit or auto-signing is permitted.

- The card fetches the latest version of the PSBT from the relay, merges its new partial signature into the existing PSBT using the BIP-174 combiner algorithm, and uploads the merged result.

- This merge-on-upload pattern prevents race conditions when two signers act simultaneously.

**Threshold and broadcast**

- When the third signature is detected in the relay, the card that added the third signature finalises the PSBT, extracts the complete signed transaction, and broadcasts it to the Bitcoin network via the SPV node\'s Wi-Fi connection.

- All subscribed cards receive the broadcast confirmation event and display a Transaction Broadcast screen.

- The SPV node monitors for block inclusion and notifies each card when the transaction is confirmed.

**5. Screen and UI specifications**

The card UI must support exactly four treasury-specific screens. All other treasury logic is handled in firmware with no user-visible intermediate states.

|                     |                                                 |                                                                                                                          |
|---------------------|-------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| **Screen**          | **Trigger**                                     | **Required elements**                                                                                                    |
| Treasury Setup      | User selects New Treasury or imports descriptor | Progress indicator (n of 5 invites), treasury address display, confirm/abort                                             |
| Pending Transaction | Card receives PSBT via Nostr subscription       | Transaction count badge, amount, destination address preview                                                             |
| Sign and Verify     | User opens a pending transaction                | Full destination address (scrollable), amount, change address, change amount, fee, sig count, Sign button, Reject button |
| Broadcast Confirmed | Third signature detected                        | Transaction ID, destination, amount, estimated confirmation time                                                         |

> *The Sign and Verify screen is the most security-critical screen in the module. The destination address must be displayed in full --- never truncated. The change address must also be shown in full. A signer who cannot verify both addresses is being asked to sign blind.*

**6. Security requirements**

The following requirements are mandatory for version 1.0. None are optional. A build that does not satisfy all of these requirements must not be released.

**6.1 Firmware integrity**

- All firmware updates must be signed with Wesatoshis\' release signing key.

- The card must verify the firmware signature against a public key burned into the secure element during manufacture before applying any update.

- Unsigned or incorrectly-signed firmware must be rejected and must not be applied under any circumstances.

- The firmware build process must be reproducible: a third party given the source code must be able to produce a binary that matches the released binary bit-for-bit.

**6.2 Key security**

- Private keys are held exclusively in the card\'s secure element and are never exported under any circumstances.

- The XPUB export (Treasury Invite) contains only public key material. No private or seed material is derivable from it.

- The treasury descriptor contains only public key material.

- PIN entry is required before any treasury operation. The PIN must be rate-limited with an exponential backoff.

- After a configurable maximum number of failed PIN attempts (default: 10), the card must wipe all key material and treasury state.

**6.3 PSBT and transaction security**

- Every signing card must independently perform SPV verification of the PSBT\'s inputs before signing. A card must not sign a transaction whose inputs it cannot verify as unspent.

- Every signing card must verify that the PSBT has not been mutated since the proposer published it. This is achieved by comparing the transaction outputs (address and amount) against what was announced via the human coordination channel.

- The SPV node\'s block headers must be no more than 24 hours stale before any signing operation. If headers are older, the card must re-sync before proceeding.

- The card must implement BIP-174 PSBT combiner logic on upload. It must fetch, merge, then upload --- never replace --- to prevent race-condition data loss.

- Replay protection: the card must reject a PSBT whose input UTXOs have already been spent in a confirmed transaction.

**6.4 Communication security**

- All PSBT payloads transmitted via Nostr must be encrypted using NIP-44 (ChaCha20-Poly1305 with a per-message nonce). The relay operator must not be able to read transaction contents.

- All connections from the card to Nostr relays must use TLS 1.2 or higher.

- The card must validate the TLS certificate of the relay endpoint. Certificate pinning is recommended for the default Wesatoshis-hosted relay.

- The treasury Nostr keypair must be derived deterministically from the treasury descriptor hash using HKDF, so all five cards independently arrive at the same keypair without out-of-band key exchange.

**6.5 Supply chain and attestation**

- On first boot, the card must display its firmware version and a truncated firmware hash. The user should be encouraged to verify this hash against the published release.

- Cards must ship with a tamper-evident physical seal. A broken seal is grounds for refusing to use the device.

- The reproducible build process (6.1) is the primary supply chain mitigation. Document it prominently in the release notes.

**7. Threat model summary**

The following table summarises the primary threats, their severity, and their mitigations. This is not an exhaustive security audit --- an independent security review is required before mainnet deployment with significant funds.

|                                     |              |                                                                                                 |
|-------------------------------------|--------------|-------------------------------------------------------------------------------------------------|
| **Threat**                          | **Severity** | **Mitigation**                                                                                  |
| Card theft                          | High         | PIN protection + wipe after N failures. Single card = 0 of 3 required signatures.               |
| Firmware tampering                  | Critical     | Signed firmware updates, secure element verification, reproducible builds.                      |
| Swap attack (output substitution)   | Critical     | Full address displayed on every signing card. SPV verifies outputs. Human confirmation policy.  |
| Stale SPV headers                   | High         | Mandatory re-sync if headers older than 24 hours. Block any signing until synced.               |
| PSBT relay mutation                 | High         | NIP-44 encryption. Each card independently verifies outputs before signing.                     |
| Race condition (concurrent signers) | Medium       | BIP-174 combiner: fetch-merge-upload, never replace.                                            |
| Replay attack                       | Medium       | Reject PSBT with already-spent UTXOs. Bitcoin UTXO model provides natural protection.           |
| Relay outage                        | Medium       | Minimum two relay endpoints. PSBT stored locally on card for manual recovery.                   |
| Social engineering                  | Medium       | Human coordination policy: verbal confirmation before signing. Card cannot enforce this.        |
| Coercion / duress                   | Medium       | 3-of-5 threshold requires coordinated coercion. Operational security for key holder identities. |
| Supply chain compromise             | High         | Reproducible builds, tamper-evident packaging, first-boot hash display.                         |
| Side-channel (power/timing)         | Low (v1)     | Documented known limitation. Mitigate in v2 with constant-time signing operations.              |

**8. Version 1.0 acceptance criteria**

The following criteria must all be satisfied before the module is released. Any criterion marked MUST that is not satisfied is a blocking defect.

|                             |                                                                                                                               |
|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| **Criterion**               | **Requirement**                                                                                                               |
| Treasury setup              | MUST: Five cards enrol into a single treasury with no laptop. All five display the same P2WSH address at ceremony completion. |
| Geographic distribution     | MUST: All three signing operations complete with cards on different Wi-Fi networks in different locations.                    |
| Unsigned firmware rejection | MUST: A modified firmware image not signed by the release key is rejected and not applied.                                    |
| Full address display        | MUST: Destination and change addresses are displayed in full (not truncated) on the Sign and Verify screen.                   |
| SPV verification            | MUST: A PSBT with a double-spend input (already-spent UTXO) is detected and rejected before signing.                          |
| Stale header detection      | MUST: A card with block headers older than 24 hours refuses to sign and prompts re-sync.                                      |
| Race condition safety       | MUST: Two cards signing simultaneously produce a valid merged PSBT, not a corrupted one.                                      |
| Relay failover              | MUST: If the primary relay is unreachable, the card falls back to the secondary relay automatically.                          |
| PIN lockout                 | MUST: After 10 failed PIN attempts, the card wipes all treasury and key material.                                             |
| Broadcast and confirmation  | MUST: The card that adds the third signature broadcasts the transaction. All five cards display the broadcast event.          |
| No private key export       | MUST: No test, debug, or production path permits private key material to leave the secure element.                            |
| Reproducible build          | SHOULD: A third party can reproduce the firmware binary from source and verify the hash matches the release.                  |

**9. Out of scope for version 1.0**

- Timelock transactions (CSV/CLTV) --- future treasury governance feature

- More than one active treasury per card

- Treasury member replacement without full re-enrolment

- Tor relay connections for traffic analysis resistance

- Threshold above or below 3-of-5 (configurable threshold is a v2 feature)

- Hardware security module (HSM) integration for institutional deployments

- Rootstock (RSK) multi-sig contracts

- Lightning Network multi-party channels

- Independent security audit --- required before deployment of funds above a defined threshold

> *An independent firmware security audit by a qualified Bitcoin security firm is strongly recommended before this module is used to custody significant funds. The threat model in section 7 is a design-time analysis, not a substitute for adversarial review.*

**10. Glossary**

|                          |                                                                                                                                                                            |
|--------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Term**                 | **Definition**                                                                                                                                                             |
| BIP-174 / PSBT           | Partially Signed Bitcoin Transaction. A standard format for passing an unsigned or partially-signed transaction between signers.                                           |
| BIP-32 XPUB              | Extended public key. A public key plus chain code that allows derivation of all child public keys in a wallet. Contains no private key material.                           |
| BIP-380 Descriptor       | A standardised string encoding all information needed to reconstruct a wallet\'s addresses. For a multi-sig treasury, it encodes all five XPUBs and the signing threshold. |
| BIP-48                   | Derivation path standard for hardware wallet multi-signature wallets.                                                                                                      |
| BIP-67                   | Standard for lexicographic sorting of public keys in multi-signature scripts, ensuring all parties derive the same script independently.                                   |
| BIP-157/158              | Compact block filter protocol. Allows SPV nodes to query for relevant transactions without revealing which addresses they are interested in.                               |
| NIP-44                   | Nostr Implementation Possibility 44. Encrypted direct messages using ChaCha20-Poly1305.                                                                                    |
| P2WSH                    | Pay-to-Witness-Script-Hash. The native SegWit output type used for multi-signature wallets.                                                                                |
| Setup Ceremony           | The step in treasury onboarding where all five card holders verify that they see the same treasury address on their screens.                                               |
| SPV node                 | Simplified Payment Verification node. Downloads block headers and uses Merkle proofs to verify transactions without storing the full blockchain.                           |
| Treasury Invite          | The XPUB string generated by a card during setup, shared with other board members to enrol them in the treasury.                                                           |
| wsh(sortedmulti(3,\...)) | The output descriptor expression for a 3-of-5 native SegWit multi-signature policy.                                                                                        |

*End of specification --- Version 0.1 Draft*
