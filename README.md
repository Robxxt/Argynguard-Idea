# Anchor: Secure Offline Communication Protocol
**Design Document & Security Whitepaper v1.0**

---

## 1. Executive Summary
**Anchor** is a high-security, offline-first mobile application designed to establish and maintain a "Web of Trust" between peers. In an era of deep fakes and AI-driven impersonation, Anchor relies on physical proximity and cryptographic proofs to validate identity.

By utilizing NFC for "Zero-Knowledge" key exchanges and strict local storage protocols, Anchor ensures that users can verify who they are communicating with—whether over a phone call, encrypted chat, or video link—without relying on central servers or internet connectivity.

---

## 2. Core Philosophy & Constraints

### 2.1 The "Zero-Trust" Model
The application assumes the network is compromised. Therefore, all trust is derived from:
1.  **Physical Proximity:** Keys can only be generated when devices physically touch (NFC).
2.  **Peer-to-Peer:** No central server holds user keys or identity graphs.

### 2.2 The "PeepHole" Visibility Protocol
To prevent "Shoulder Surfing" or "Lunch Break" attacks (where an attacker briefly accesses an unlocked phone), the application enforces strict visibility rules:
* **No List Views:** Users can never see the full list of future verification words.
* **One-at-a-Time:** Users can only view the *current* active key.
* **Burn-on-Sight:** Once a key is revealed/used, it is cryptographically erased and the pointer moves to the next. It is impossible to look back.

### 2.3 Fail-Secure Revocation
If a contact is suspected of being compromised, the system favors security over convenience. Revocation immediately wipes local keys, forcing a new physical meeting to re-establish trust.

---

## 3. Security Architecture

### 3.1 The "Vault" Storage
* **Encryption at Rest:** All sensitive data is stored in a local database (`Isar` or `sqflite`) encrypted with **AES-256-GCM** via SQLCipher.
* **Key Management:** The database decryption key is **never** stored in plain text.
    * It is derived from the user's **Master Password** + **Salt**.
    * It is wrapped by the device's Hardware Security Module (iOS Secure Enclave / Android Keystore).
* **Volatile Memory:** Cryptographic buffers (especially the word lists) are zero-filled immediately after use.

### 3.2 The "Gatekeeper" Authentication
Access to the app requires passing a two-tier check:
1.  **Tier 1 (Knowledge):** A strong Alphanumeric Password or 6-digit PIN (set at install).
2.  **Tier 2 (Possession/Inherence):** Biometric authentication (FaceID/Fingerprint) is required to unlock the Vault for daily use.
    * *Fallback:* After 3 failed biometric attempts, the app strictly requires the Master Password.

### 3.3 Anti-Forensics
* **Root/Jailbreak Detection:** The app will refuse to launch if the OS integrity is compromised.
* **Screen Shield:** The app view is obscured (blurred) in the OS "Recent Apps" switcher.
* **Screenshot Defense:** (OS dependent) Attempting to take a screenshot of a "Secret Word" triggers a security warning or blacks out the capture.

---

## 4. Data Models

### 4.1 User Identity (`User`)
*The local device owner.*
```json
{
  "uuid": "String (UUIDv4)",
  "publicKey": "String (Base64 - Ed25519 or similar)",
  "privateKey": "EncryptedString (Stored in Secure Enclave)",
  "mfaHash": "String (Argon2id hash of Master Password)",
  "created_at": "Timestamp"
}
```

### 4.2 Peer Contact (`Peer`)

*A trusted contact established via NFC.*
```json
{
  "uuid": "String (UUIDv4 - Derived from their Public Key)",
  "displayName": "String (User assigned, e.g., 'Alice')",
  "publicKey": "String (Base64)",
  "trustLevel": "Enum [VERIFIED, COMPROMISED, ARCHIVED]",
  "lastInteraction": "Timestamp",
  "notes": "EncryptedString (Optional user notes)"
}
```

### 4.3 OTP Bundle (OtpBundle)

*The consumable "fuel" for verification. This contains the shared secrets.*

```json
{
  "sessionId": "String (Unique ID for this specific handshake)",
  "peerId": "Link -> Peer.uuid",
  "totalKeys": "Integer (Fixed at 50)",
  "currentIndex": "Integer (0 to 49)",
  "encryptedBlob": "Binary (AES-256 encrypted list of 50 word-pairs)"
}
```

*Note: The encryptedBlob is opaque. It is only decrypted into RAM momentarily when the specific Verification View is active.*

## 5. Protocols & Algorithms
### 5.1 The NFC Handshake (`Key Generation`)

*Goal: Create a shared set of 50 word-pairs without a third party.*

1. **Initiation**: Users A and B enter "Handshake Mode."
2. **Exchange**: Devices touch. An Ephemeral Elliptic Curve Diffie-Hellman (ECDH) exchange occurs over NFC.
3. **Derivation**: Both devices derive a Shared Secret (S).
4. **Expansion**: A Key Derivation Function (HKDF) expands S into 50 unique integers.
5. **Mapping**: Integers are mapped to the PGP Word List (Biometric word lists designed for distinct pronunciation).
6. **Storage**: The list is saved to the OtpBundle, and S is wiped from memory.

### 5.2 The "Bouncer" Import Protocol

*Goal: Prevent Denial of Service (DoS) or corruption during data restore.*

1.  **Header Check**: Verify file magic bytes.
2.  **Size Gate**: Reject file > 50MB.
3.  **Integrity Check**: Calculate HMAC-SHA256 of the payload. If it mismatches the signature, reject.
4.  **Stream Parsing**: Use a SAX-style JSON parser to prevent memory overflows from deeply nested objects.

### 5.3 The "Hard Break" Revocation

Goal*: Ensure security when trust is lost.*

1.  **Action**: User taps "Revoke" on a contact.
2.  **System Response**: The app permanently deletes the OtpBundle for that peer.
3.  **Result**: Synchronization is broken. Even if the other peer has the keys, the local user has nothing to compare them against. Communication is effectively blocked until a new NFC handshake occurs.

## 6. UI/UX Design System

### 6.1 Visual Aesthetics

**Theme**: "Dark Mode First" (Cyber-Security aesthetic).

**Typography**:
  - **UI Text**: Inter or Roboto (Clean, Sans-Serif).
  - **Secret Words**: JetBrains Mono (Monospaced for unambiguous reading).

**Palette**:
  - **Background**: Deep Slate (`#0F172A`)
  - **Text**: Pale Silver (`#E2E8F0`)
  - **Success**: Neon Green (`#00E676`)
  - **Danger**: Crimson Red (`#FF1744`)

## 6.2 View Hierarchy

A. The Gatekeeper (Login)

    Minimalist screen.

    Biometric prompt appears immediately.

    Fallback to PIN/Password after failure.

B. The "Safe List" (Home)

    Top Bar: Search ("Find Contact..."), Settings (Backup/Restore).

    List: Peers sorted by recent interaction.

        Indicators: Green Dot (Safe), Orange Dot (<5 Keys), Red Dot (Revoked/Empty).

    FAB: Large + / NFC icon to add or renew keys.

C. Verification Mode (The "Game")

    Context: Opened by tapping a Peer.

    State 1 (Covered):

        Large blurred card.

        Text: "Key #44 ready. Tap and hold to reveal."

    State 2 (Revealed - Long Press):

        Card un-blurs. Displays: "BATTERY".

        Below: "Ask [Name] for their word."

        Options: [CORRECT] | [WRONG] | [SKIP].

    State 3 (Result):

        Correct: Animation of a lock snapping shut. Index moves to #45.

        Wrong: Warning modal: "Mark session compromised?".

D. Contact Details (Admin)

    Accessible via "Settings" icon inside Verification Mode.

    Actions:

        Edit Name.

        Regenerate Keys: (Grayed out unless NFC active).

        Revoke Trust: (Red Zone).

## 7. Technology Stack

- **Framework**: Flutter (Dart)
- **Database**: Isar (NoSQL) + SQLCipher Encryption
- **Secure Storage**: flutter_secure_storage (Keychain/Keystore)
- **Biometrics**: local_auth
- **NFC**: nfc_manager
- **Cryptography**: pointycastle or cryptography (Dart FFI)

## 8. Future Roadmap

- **Post-Quantum Cryptography**: migrating from ECDH to Kyber/Dilithium algorithms once mobile libraries mature.
- **Duress PIN**: A "fake" PIN that wipes the database instantly if the user is forced to unlock the phone at gunpoint.
- **Wearable Integration**: Viewing the verification word directly on a paired Smart Watch for discreet checking.
