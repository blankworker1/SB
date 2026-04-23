**WESATOSHIS LABS**

**Sovereign Boardroom**

Firmware Module Specification

*k-of-N Geographically Distributed Multi-Signature Treasury*

Version: 0.3 Draft

Classification: Confidential --- Internal Technical

Date: April 2026

Target hardware: ESP32-S3 device with microSD slot and Wi-Fi (reference: Waveshare ESP32-S3-LCD-1.47). Wesatoshis Card integration planned for future release.

**1. Purpose and scope**

This document specifies the Sovereign Boardroom firmware --- an open-source, hardware-agnostic Bitcoin multi-signature treasury tool distributed as a bootable microSD card image. The firmware runs on any compatible ESP32-S3 device with a display, microSD slot, and Wi-Fi. The reference development hardware is the Waveshare ESP32-S3-LCD-1.47 board. Future integration with the Wesatoshis Card is planned but not required for the proof of concept or version 1.0 release.

The confirmed aim of this specification is:

> *To construct an open-source, hardware-agnostic multi-signature Bitcoin treasury platform that allows k-of-N device holders in different geographic locations to build, coordinate, and sign Bitcoin transactions using only their Sovereign Boardroom devices --- a microSD card inserted into any compatible ESP32-S3 hardware --- and a Nostr-compatible messaging application on their mobile phones. The threshold k and the total signer count N are configured by any one device holder during the one-time setup ceremony and are immutable for the lifetime of that treasury. Any device holder may initiate a transaction at any time. The firmware assigns no privileged roles. Governance decisions --- who proposes transactions, who holds which device --- are made outside the device by the organisation.*

This module is scoped to Bitcoin mainnet transactions using P2WSH (native SegWit) multi-signature outputs. Rootstock (RSK) and Lightning Network extensions are out of scope for version 1.0.

**2. Context and design philosophy**

Existing multi-signature treasury solutions require one of two compromises: either a general-purpose computer running coordinator software (Sparrow, Specter Desktop), or trust in a custodial third party (Unchained, Casa). Neither option is acceptable for a community treasury that prioritises censorship resistance, physical mobility, and air-gap-equivalent security.

The product is the microSD card, not the hardware. The firmware binary, encrypted seed, treasury descriptor, SPV headers, and all session state reside entirely on the SD card. Any compatible ESP32-S3 device is a valid vessel. This architecture means the tool can be distributed by post, requires no proprietary hardware, and allows future retargeting to the Wesatoshis Card or any other ESP32-S3 platform by updating only the hardware pin configuration file.

**2.1 Guiding principle --- equal responsibility**

The Sovereign Boardroom firmware assigns no roles and recognises no hierarchy among device holders. Every device in the treasury is equal. Any device holder may initiate a transaction. Any device holder may sign a transaction. Any device holder may assemble the treasury descriptor during the setup ceremony. The firmware has no concept of administrator, treasurer, CEO, or designated proposer.

This is a deliberate design decision. Roles such as treasurer, proposer, or coordinator are governance decisions made by the organisation that holds the treasury. They are enforced by human agreement and operational procedure, not by the firmware. The firmware's job is to make cryptographic operations possible; the organisation's job is to decide who does what and when.

The consequence of equal capability is equal responsibility. Every device holder must understand that:

- Their device is a full signing instrument. It can initiate and authorise real Bitcoin transactions. There is no safety net built into the firmware beyond the k-of-N threshold.

- They are responsible for verifying every transaction they sign. The device displays the full destination address, amount, change address, and fee. Signing without reading these details is a security failure.

- They should not sign a transaction they were not expecting. If a pending transaction appears on the device that no member of the organisation has communicated via the human coordination channel, it should not be signed until the group has discussed it.

- Their SD card and PIN must be protected with the same care as cash. The device is stateless hardware. The SD card is the wallet. Loss of the card to an attacker who also obtains the PIN compromises that keyholder's share of the treasury.

- Changing the treasury structure --- the k-of-N threshold, the set of keyholders, or the treasury address --- requires a new setup ceremony and a new wallet. This is not a limitation; it is the mechanism that makes the existing treasury's rules permanent and tamper-proof.

*These responsibilities must be communicated to all N device holders before the setup ceremony begins. The firmware cannot enforce them. The organisation must.*

**2.2 Target user profile**

The target user is a competent Bitcoin self-custodian who:

- Understands self-custody and holds a single-signature cold wallet

- Has heard of multi-signature wallets but has no practical experience creating or signing a multi-sig transaction

- Uses a Lightning wallet for day-to-day transactions

- Communicates with their board via Nostr or a Nostr-compatible messenger

The module must make the complexity of multi-signature coordination completely invisible. Technical terms such as XPUB, PSBT, and descriptor must not appear in the user interface.

**2.3 Non-goals for version 1.0**

- Speed optimisation --- correctness and security take priority over transaction throughput

- Lightning Network channel management

- Rootstock / RSK smart contract interaction

- Mobile companion app --- Nostr messaging apps (Damus, Amethyst, Primal) serve this role

- More than one active treasury per card --- a single treasury per card is sufficient for v1

**3. System architecture**

The Sovereign Boardroom is a bootable microSD card image. The hardware is a commodity vessel. All firmware, cryptographic state, and treasury data reside on the SD card. Inserting the card into any compatible ESP32-S3 device and applying power starts the tool. This section describes the SD card layout, the hardware configuration model, the session lifecycle, and the firmware hash verification mechanism.

**3.1 SD card layout**

The SD card is formatted FAT32 and contains the following directory structure. No data outside this structure is written by the firmware.

/boot/firmware.bin

> Compiled ESP32-S3 firmware binary. Loaded and executed on boot. Never modified at runtime.

/boot/firmware.sha256

> 64-character SHA-256 hash of firmware.bin, written at release time. Read on boot and compared against the computed hash of the binary. Mismatch halts boot with a red warning screen.

/boot/RELEASE.txt

> Human-readable version string, GitHub release tag, full SHA-256 hash, and release date. Displayed on the device status screen.

/config/hardware.json

> GPIO pin assignments for display, SD card, and Wi-Fi for the target hardware. Changing this file retargets the firmware to different hardware without recompilation. Ships pre-configured for the Waveshare ESP32-S3-LCD-1.47.

/config/wifi.json

> List of known Wi-Fi SSIDs and passwords. Typically contains the board member\'s phone hotspot. Tried in order on each session. Written during setup ceremony.

/config/relays.json

> Two Nostr relay endpoints (primary and fallback). Written during setup ceremony.

/treasury/descriptor.json

> The wsh(sortedmulti(k,\...)) descriptor for this treasury. Contains k, N, all N XPUBs, and the derived P2WSH address. Written once at setup ceremony completion. Never overwritten.

/treasury/seed.enc

> BIP-39 seed encrypted with a key derived from the user\'s PIN via PBKDF2. Decrypted in RAM at runtime only. Plaintext seed is never written to SD at any point.

/treasury/spv_headers.bin

> Compact block filter headers (BIP-157/158), append-only. Incrementally updated each session from the last known tip. Never rewritten from genesis.

/session/psbt_pending.psbt

> The current in-flight PSBT. Updated after each signing round. Cleared after broadcast confirmation.

/session/broadcast_log.json

> Transaction IDs of all broadcast transactions. Used for replay protection. Append-only.

Remove the SD card and the hardware device is blank. All treasury state is on the card. The card without the PIN is useless. The PIN without the card is useless.

**3.2 Session lifecycle**

Each use of the device is a discrete session. There is no background process and no persistent network connection between sessions. Wi-Fi is active only during the session itself.

- Insert SD card. Connect USB power bank. Device boots automatically.

- Firmware verifies its own SHA-256 hash against firmware.sha256. Mismatch halts with red warning. Match proceeds.

- Home screen displays version number and first 12 characters of firmware hash. User verifies against GitHub releases page on phone.

- User enters PIN. Seed is decrypted in RAM. Plaintext seed exists only in memory for the duration of the session.

- Wi-Fi activates. Device connects to known hotspot from wifi.json. Syncs SPV headers incrementally from last known tip.

- Device fetches pending PSBT events from Nostr relay. If none: idle screen. If pending: full transaction details displayed for review and signing.

- If threshold met after signing: device broadcasts transaction and displays mempool.space QR code on screen.

- User scans QR with phone to confirm mempool receipt. Unplugs USB power. Session ends. RAM cleared.

**3.3 Firmware hash verification**

The firmware hash mechanism protects against an attacker who gains physical access to an SD card and replaces firmware.bin with a malicious binary. The mechanism has two layers.

**Automatic check (device).** On every boot the device computes SHA-256 of firmware.bin and compares it to the hash stored in firmware.sha256. If they do not match the device displays a full-screen red warning and refuses to proceed to PIN entry. The seed is never decrypted.

**Manual check (user).** The home screen permanently displays the firmware version tag (e.g. v1.0.2) and the first 12 characters of the SHA-256 hash (e.g. a3f8c12d4e91). The user opens the GitHub releases page on their phone, locates the matching version tag, and confirms the hash prefix matches. This check takes under 30 seconds and should be performed before every signing session.

**Independent verification (advanced).** Because the project is open source with reproducible builds, any board member can clone the repository, compile the tagged release from source, and verify that the binary they produce has the same SHA-256 hash as the published release. This is the highest level of assurance and is encouraged for any treasury holding significant funds.

**3.4 k-of-N configuration**

The threshold k and total signer count N are set by any one device holder during the setup ceremony using a number picker on the device touchscreen. They are not hardcoded. Valid ranges are N: 2 to 15, k: 1 to N. The firmware warns if k is less than or equal to N divided by 2 (below majority threshold) but does not block the user from proceeding. The device holder who runs the setup ceremony has no elevated privileges after it is complete.

Once confirmed, k and N are embedded in the descriptor written to descriptor.json. They cannot be changed without generating a new descriptor --- which means a new treasury address, a new setup ceremony, and moving funds from the old address to the new one. This immutability is intentional: it eliminates configuration drift and makes the governance decision permanent and auditable.

**3.5 Open source model and release process**

The Sovereign Boardroom firmware is published on GitHub under an open-source licence. The full source code, build scripts, and release notes are publicly available. Every release follows this process:

- Source code is tagged on GitHub with the version number (e.g. v1.0.2).

- The binary is compiled from that exact tagged commit using the reproducible build toolchain.

- SHA-256 of the binary is computed and published in the GitHub release notes alongside the binary download.

- The SD card image (firmware.bin + firmware.sha256 + RELEASE.txt + default config files) is published as a downloadable ZIP. Users flash the SD card from this image.

- Any user can independently reproduce step 2 and verify that the published binary matches the source code. Instructions for doing so are included in the repository README.

**3.6 Component overview**

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

This phase is completed once, when the treasury is first created. It is repeated only when the treasury structure changes --- a new keyholder, a different threshold, or a deliberate migration to a new address. All N device holders should be present or reachable before the ceremony begins.

**Step 0 --- First boot and key share generation**

This step happens on every device independently, before any communication between holders. It is the most important step in the entire process. Each device holder performs it alone, in private.

- Device holder inserts a blank SD card loaded with the Sovereign Boardroom firmware image and applies USB power. The device boots into first-run setup mode.

- The device generates a fresh BIP-39 seed using the hardware entropy source. This seed is the device holder's key share. It is generated once and only once. It cannot be regenerated, recovered from the firmware, or reconstructed from any other card in the treasury.

- The device prompts the holder to set a PIN. The seed is immediately encrypted using a key derived from this PIN via PBKDF2 and written to seed.enc on the SD card. The plaintext seed is erased from RAM. From this point forward, the plaintext seed only exists in RAM during an active session when the correct PIN is entered.

- The device displays a confirmation screen: "Your key share has been created and encrypted. This SD card is your signing instrument. The key share cannot be recovered if this card is lost without a backup."

> *The irreversibility of key share generation is a security feature, not a limitation. A seed that can be regenerated or recovered by a third party is a seed that can be stolen by a third party. Each key share is sovereign to the device holder who created it and to no one else. This is what makes the treasury genuinely self-custodial.*

**Step 1 --- Generate and share invite**

- Device holder opens the Treasury section. The key share generated in Step 0 is already on this card. Step 1 derives only the public portion of that key --- the XPUB --- which is safe to share with the other holders.

- Card derives the member\'s XPUB at the BIP-48 path and encodes it alongside the derivation path as a compact string (the Treasury Invite).

- The invite string is displayed on screen and can be copied into any Nostr messenger for distribution to the other N-1 device holders.

- No private key material is included in or derivable from the invite string.

**Step 2 --- Assemble the descriptor**

- Any one device holder collects all N invite strings. This person has no privileged role after the setup ceremony is complete. The role is temporary and practical, not structural.

- Their card parses all N XPUBs, sorts keys per BIP-67, constructs the wsh(sortedmulti(k,\...)) output descriptor, and derives the treasury P2WSH address.

- The descriptor string is shared back to the group via Nostr.

**Step 3 --- Import and verify**

- Each of the remaining N-1 devices imports the descriptor string.

- Every card independently derives the treasury address from the descriptor and displays it on screen.

- All N device holders verbally or via Nostr confirm they see the same address. This is the Setup Ceremony. The address is the proof that every key share has been correctly contributed and that the treasury is ready.

- If any card displays a different address, setup fails loudly and must restart. This indicates a tampered descriptor.

> *The Setup Ceremony is the single most important security moment in the entire workflow. The address displayed on all N devices must match exactly before any funds are deposited.*

**Step 4 --- Backup the SD card before depositing funds**

After the address ceremony is complete, the device displays a mandatory reminder screen before the treasury is marked active. No funds should be deposited until this step is complete for every device holder.

> **BACKUP REQUIRED BEFORE USE**
>
> This SD card is your key share.
>
> If lost with no backup, your share
>
> cannot be recovered.
>
> **Before depositing funds:**
>
> 1\. Copy this SD card to a second card
>
> 2\. Store backup in a separate location
>
> 3\. Never store card and PIN together

The backup card is a byte-for-byte copy of the original, including the encrypted seed. It is protected by the same PIN. An attacker who obtains the backup card without the PIN cannot decrypt the seed. The backup does not create a new key share or a new participant in the treasury. It is a recovery mechanism against physical loss or damage only.

The device requires the holder to tap a confirmation button labelled "I have made a backup" before the treasury is marked active. This is the final gate before the treasury address can receive funds. The firmware cannot verify that a backup was actually made --- this confirmation is the holder's personal commitment.

> *Loss of a key share without a backup does not necessarily mean loss of treasury funds. If the remaining N-1 holders can still form a quorum of k signatures, they can move funds to a new treasury address. The correct response to a lost card with no backup is: form a quorum, move the funds, conduct a new setup ceremony with a replacement device holder. This is why choosing a k threshold that leaves headroom --- 3-of-5 rather than 5-of-5 --- is a resilience decision as much as a governance one.*

**4.2 Phase 2 --- Transaction proposal**

Any device holder may propose a transaction at any time. The firmware assigns no designated proposer. Which device holder initiates a given transaction is an out-of-device decision made by the organisation.

- The initiating device holder opens the Treasury app and selects Send.

- They enter the destination address and amount using the card\'s input interface.

- The card constructs a PSBT: it selects UTXOs from the treasury address, sets the destination output, calculates the change output back to the treasury address, and applies a fee rate based on current mempool data from the SPV node.

- The card adds the first partial signature (sig \#1 of 3 required).

- The card performs SPV verification: it confirms the input UTXOs exist and are unspent using BIP-157/158 compact filters.

- The PSBT is published as a NIP-44 encrypted event to the treasury Nostr keypair.

- The card displays a confirmation screen showing the transaction ID, destination address (full, untruncated), amount, change amount, and fee.

> *The device holder who initiates the transaction should communicate the details to the other N-1 holders via a separate human channel (Nostr group message, voice call) before others sign. The firmware cannot enforce this policy. It is the shared responsibility of all N device holders to verify transaction details out-of-band before signing. No device holder should sign a transaction they were not expecting or have not independently verified.*

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

The requirements in sections 6.1 through 6.5 are mandatory for version 1.0. None are optional. A build that does not satisfy all of them must not be released. Section 6.0 below is not a list of requirements but a precise statement of the security properties the architecture provides and the limits of those properties. It must be read before the requirements are implemented.

**6.0 Private key isolation --- the core security property**

The private key never exists on a network-connected general-purpose computer at any stage of the signing process. It exists only in the RAM of a single-purpose device during the narrow window of an active signing session. This is the architectural property that defines the security of the Sovereign Boardroom and distinguishes it from every software wallet and every web-based multisig coordinator.

**What signs a transaction.** When a device holder enters their PIN and confirms a transaction, the firmware derives a private key from the decrypted seed, uses it to sign the hash of the transaction using ECDSA on the secp256k1 curve, and adds the resulting signature to the PSBT. The derived key is then discarded from RAM. What leaves the device is the signature --- a mathematical proof that the private key was used --- not the key itself. This is a one-way operation. A valid signature cannot be reversed to reveal the private key that produced it. This is the fundamental asymmetry of elliptic curve cryptography on which all of Bitcoin's security is based.

**What is and is not on the network.** The PSBT travels over the network encrypted via Nostr. The broadcast transaction travels over the Bitcoin peer-to-peer network in the clear, as all Bitcoin transactions do. Neither contains private key material. The Nostr relay sees only encrypted blobs. The Bitcoin network sees a fully signed transaction. Neither has any path to the private key. The private keys are not in the protocol, the relay, or the network at any point in the process.

**What this eliminates.** The majority of cryptocurrency theft occurs through remote attacks: malware on a general-purpose computer, browser extensions, clipboard hijacking, phishing for seed phrases, compromised coordinator websites, or exchange hacks. All of these attacks require the private key to exist, even briefly, on a networked general-purpose machine. Because the Sovereign Boardroom device has no general-purpose OS, no browser, no background processes, and no persistent network connection, this entire class of remote attack has no surface to work against. A remote attacker cannot reach the key because the key is never on a network.

**Even all N holders together cannot expose a private key.** If all N device holders assembled in the same room simultaneously, there is no operation they could collectively perform that would expose any individual's private key. The keys never aggregate. What combines are the partial signatures, not the keys themselves. The treasury address is controlled by a mathematical relationship between N independent keys that were generated separately, stored separately, and used separately. The keys remain permanently isolated on their respective SD cards.

**The honest limits of this property.** Private key isolation is not the same as being impervious to all attacks. The following attack vectors remain real and must be acknowledged honestly.

- **Firmware bugs.** A memory management error, a debug logging statement left in production code, or a buffer overflow in the signing path could expose key material. The architecture minimises the firmware attack surface --- single-purpose code, no OS, no peripherals beyond what is needed --- but it does not make firmware bugs impossible. Open source code and reproducible builds exist precisely so the community can audit for and report such bugs before they cause harm.

- **Weak PIN.** The seed is encrypted with a PBKDF2-derived key from the PIN. A weak PIN --- four zeros, a birthday, a repeating digit --- can be brute-forced offline by an attacker with physical possession of the SD card. The firmware enforces a minimum PIN length and warns against obvious patterns. The PIN is the single most important decision the device holder makes. It is the only path by which a remote or opportunistic attacker could extract the private key without the holder's active participation.

- **Physical access during a live session.** During the window between PIN entry and session end, the decrypted seed and derived keys exist in RAM. A sophisticated attacker with physical access to the powered device could in theory perform a cold boot attack --- rapidly cooling RAM chips to preserve their state after power loss, then reading the contents. This requires specialised equipment and close physical proximity. It is not a practical threat for community treasury use cases but it is a real technique and it is honest to acknowledge it.

- **Side-channel attacks.** Power consumption and electromagnetic emissions during ECDSA signing can leak information about the private key to an attacker with measurement equipment and physical proximity. The ESP32-S3 does not have dedicated hardware countermeasures against power analysis of the same calibre as a purpose-built secure element. This is a nation-state level capability, not a practical threat for the intended use case, but it is the honest reason why the word "unhackable" does not appear in this specification.

- **Coercion.** No cryptographic system protects against a device holder being compelled to sign under physical threat. The k-of-N threshold provides structural resilience --- coercing one holder is not enough --- but it is not unlimited. The operational security of keyholder identities is a governance matter outside the scope of the firmware.

> *The correct security claim for this product is precise: the private keys are never on a network-connected general-purpose computer at any stage of the signing process. This eliminates the entire class of remote attacks that accounts for the majority of real-world cryptocurrency theft. Physical attacks remain possible but require possession of both the device and the PIN --- which is the correct and honest security boundary for any hardware signing device.*

**6.1 Firmware integrity**

- All firmware updates must be signed with Wesatoshis\' release signing key.

- The card must verify the firmware signature against a public key burned into the secure element during manufacture before applying any update.

- Unsigned or incorrectly-signed firmware must be rejected and must not be applied under any circumstances.

- The firmware build process must be reproducible: a third party given the source code must be able to produce a binary that matches the released binary bit-for-bit.

**6.2 Key security**

- Private keys are derived in RAM during an active signing session only. They are never written to the SD card, never logged, and never transmitted. The firmware must zero all key material from RAM immediately after the signing operation completes and unconditionally when the session ends.

- The XPUB export (Treasury Invite) contains only public key material. No private or seed material is derivable from it.

- The treasury descriptor contains only public key material.

- PIN entry is required before any treasury operation. The PIN must be rate-limited with an exponential backoff.

- After a configurable maximum number of failed PIN attempts (default: 10), the card must wipe all key material and treasury state.

**6.3 PSBT and transaction security**

- Every signing card must independently perform SPV verification of the PSBT\'s inputs before signing. A card must not sign a transaction whose inputs it cannot verify as unspent.

- Every signing device must verify that the PSBT has not been mutated since it was published. This is achieved by comparing the transaction outputs (address and amount) shown on the device screen against what was communicated via the human coordination channel. Each device holder bears equal responsibility for this verification. Signing without verifying is a shared security failure, not an individual one.

- The SPV node\'s block headers must be no more than 24 hours stale before any signing operation. If headers are older, the card must re-sync before proceeding.

- The card must implement BIP-174 PSBT combiner logic on upload. It must fetch, merge, then upload --- never replace --- to prevent race-condition data loss.

- Replay protection: the card must reject a PSBT whose input UTXOs have already been spent in a confirmed transaction.

**6.4 Communication security**

- All PSBT payloads transmitted via Nostr must be encrypted using NIP-44 (ChaCha20-Poly1305 with a per-message nonce). The relay operator must not be able to read transaction contents.

- All connections from the card to Nostr relays must use TLS 1.2 or higher.

- The card must validate the TLS certificate of the relay endpoint. Certificate pinning is recommended for the default Wesatoshis-hosted relay.

- The treasury Nostr keypair must be derived deterministically from the treasury descriptor hash using HKDF, so all all N devices independently arrive at the same keypair without out-of-band key exchange.

**6.5 Supply chain and attestation**

- On first boot, the card must display its firmware version and a truncated firmware hash. The user should be encouraged to verify this hash against the published release.

- Cards must ship with a tamper-evident physical seal. A broken seal is grounds for refusing to use the device.

- The reproducible build process (6.1) is the primary supply chain mitigation. Document it prominently in the release notes.

**7. Threat model summary**

The following table summarises the primary threats, their severity, and their mitigations. This is not an exhaustive security audit --- an independent security review is required before mainnet deployment with significant funds.

|                                     |              |                                                                                                                                                                                                |
|-------------------------------------|--------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Threat**                          | **Severity** | **Mitigation**                                                                                                                                                                                 |
| Card theft                          | High         | PIN protection + wipe after N failures. Single card = 0 of 3 required signatures.                                                                                                              |
| Firmware tampering                  | Critical     | Signed firmware updates, secure element verification, reproducible builds.                                                                                                                     |
| Swap attack (output substitution)   | Critical     | Full address displayed on every signing card. SPV verifies outputs. Human confirmation policy.                                                                                                 |
| Stale SPV headers                   | High         | Mandatory re-sync if headers older than 24 hours. Block any signing until synced.                                                                                                              |
| PSBT relay mutation                 | High         | NIP-44 encryption. Each card independently verifies outputs before signing.                                                                                                                    |
| Race condition (concurrent signers) | Medium       | BIP-174 combiner: fetch-merge-upload, never replace.                                                                                                                                           |
| Replay attack                       | Medium       | Reject PSBT with already-spent UTXOs. Bitcoin UTXO model provides natural protection.                                                                                                          |
| Relay outage                        | Medium       | Minimum two relay endpoints. PSBT stored locally on card for manual recovery.                                                                                                                  |
| Social engineering                  | Medium       | Human coordination policy: verbal confirmation before signing. Card cannot enforce this.                                                                                                       |
| Coercion / duress                   | Medium       | k-of-N threshold requires coordinated coercion of k members. Higher k = stronger coercion resistance. Operational security for keyholder identities is a policy matter outside firmware scope. |
| Supply chain compromise             | High         | Reproducible builds, tamper-evident packaging, first-boot hash display.                                                                                                                        |
| Side-channel (power/timing)         | Low (v1)     | Documented known limitation. Mitigate in v2 with constant-time signing operations.                                                                                                             |

**8. Version 1.0 acceptance criteria**

The following criteria must all be satisfied before the module is released. Any criterion marked MUST that is not satisfied is a blocking defect.

|                             |                                                                                                                                                                                                                     |
|-----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Criterion**               | **Requirement**                                                                                                                                                                                                     |
| Treasury setup              | MUST: N devices enrol into a single treasury with no laptop. All N display the same P2WSH address at ceremony completion. The test must be run with at least one 2-of-3 configuration and one k-of-N configuration. |
| Geographic distribution     | MUST: All three signing operations complete with cards on different Wi-Fi networks in different locations.                                                                                                          |
| Unsigned firmware rejection | MUST: A modified firmware image not signed by the release key is rejected and not applied.                                                                                                                          |
| Full address display        | MUST: Destination and change addresses are displayed in full (not truncated) on the Sign and Verify screen.                                                                                                         |
| SPV verification            | MUST: A PSBT with a double-spend input (already-spent UTXO) is detected and rejected before signing.                                                                                                                |
| Stale header detection      | MUST: A card with block headers older than 24 hours refuses to sign and prompts re-sync.                                                                                                                            |
| Race condition safety       | MUST: Two cards signing simultaneously produce a valid merged PSBT, not a corrupted one.                                                                                                                            |
| Relay failover              | MUST: If the primary relay is unreachable, the card falls back to the secondary relay automatically.                                                                                                                |
| PIN lockout                 | MUST: After 10 failed PIN attempts, the card wipes all treasury and key material.                                                                                                                                   |
| Broadcast and confirmation  | MUST: The device that adds the k-th signature broadcasts the transaction. All N devices display the broadcast event.                                                                                                |
| No private key export       | MUST: No test, debug, or production path permits private key material to leave the secure element.                                                                                                                  |
| Reproducible build          | SHOULD: A third party can reproduce the firmware binary from source and verify the hash matches the release.                                                                                                        |

**9. Out of scope for version 1.0**

- Timelock transactions (CSV/CLTV) --- future treasury governance feature

- More than one active treasury per card

- Treasury member replacement without full re-enrolment

- Tor relay connections for traffic analysis resistance

- Threshold above 15-of-15 (the Bitcoin script limit for sortedmulti)

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
| Setup Ceremony           | The step in treasury onboarding where all N device holders verify that they see the same treasury address on their screens.                                                |
| SPV node                 | Simplified Payment Verification node. Downloads block headers and uses Merkle proofs to verify transactions without storing the full blockchain.                           |
| Treasury Invite          | The XPUB string generated by a card during setup, shared with other board members to enrol them in the treasury.                                                           |
| wsh(sortedmulti(3,\...)) | The output descriptor expression for a k-of-N native SegWit multi-signature policy. k and N are set during the setup ceremony.                                             |

**11. Open Questions for the Development Team**

Four of the six questions have been answered by the Wesatoshis development team (April 2026). Answered questions are marked **RESOLVED**. Unanswered questions are marked **OPEN**. Where an answer has a design implication, this is noted immediately below the answer.

- **Secure element capability.** Does the secure element natively support BIP-48 key derivation, and what is the maximum PSBT size it can process in working memory? A k-of-N PSBT with existing partial signatures grows with N. At maximum (15-of-15) a PSBT can approach 8 KB; if memory is constrained, a chunked signing strategy is required.

> **RESOLVED.** BIP-48 derivation is supported. Working memory has no practical limit for PSBT operations. Chunked signing strategy is not required. No architecture change needed.

- **Wi-Fi and persistent connections.** Can the Wi-Fi chipset maintain a persistent WebSocket connection without significant battery drain? If not, a pull-based polling architecture with a defined interval is required.

> **RESOLVED --- with design implication.** Wi-Fi always drains the battery regardless of connection mode. Persistent WebSocket subscriptions are not viable as a background state.
>
> **Design implication:** The card must operate in a user-initiated connection model. Wi-Fi is activated only when the user opens the Treasury app. On launch, the card connects, syncs SPV headers, fetches any pending PSBT events from the Nostr relay, and presents them to the user. Wi-Fi is then shut down. This is a pull model: the card does not receive push notifications. The Nostr phone client handles background alerting; it notifies the user when a pending transaction exists, and the user then opens their card to act. Section 5 (Phase 3, Step 3.1) must be updated to reflect this flow. The battery constraint also means SPV sync should be incremental: the card downloads only headers since its last known tip, not the full header chain, on each connection.

- **Default Nostr relay selection.** Which two Nostr relays will ship as the default configuration? The choice has implications for censorship resistance, uptime guarantees, and the setup flow. Self-hosting should be a first-class option.

> **OPEN.** Awaiting decision from Wesatoshis team. Blocking the setup ceremony UX design.

- **Screen resolution and address display.** Can the screen display a full 62-character bech32 address legibly without scrolling? If not, a scrollable address view is required and must be UX-tested to confirm users scroll to verify the complete string.

> **RESOLVED.** The screen can display a full bech32 address legibly. No scrollable address view is required for version 1. The UI requirement in Section 7 (full address, no truncation) is confirmed feasible on current hardware.

- **Treasury Nostr keypair derivation.** Should the treasury Nostr keypair be derived deterministically from the descriptor hash via HKDF (preferred), or generated randomly and distributed during setup?

> **OPEN.** Awaiting confirmation from firmware team that no cryptographic objection exists to HKDF derivation from the descriptor hash. Blocking the Phase 1 setup implementation.

- **Security audit requirement.** Is an independent hardware and firmware security audit required before public release? If so, which firm will be engaged, what is the scope, and will the report be published?

> **OPEN.** Awaiting decision from Wesatoshis leadership. A third-party audit is strongly recommended before any treasury holds significant funds. This decision affects the release timeline.

*End of specification --- Version 0.3 Draft*
