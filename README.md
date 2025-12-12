1. Executive Summary

Argynguard is an offline-first mobile security tool designed to authenticate human identity in an age of AI-driven impersonation. It creates a cryptographic "Web of Trust" between peers, allowing them to verify each other via a challenge-response game during phone calls, video chats, or messaging.

Unlike Signal or Telegram (which secure the channel), Argynguard secures the entity. It answers the question: "Is the voice on the phone actually my CEO, or an AI clone?"
2. Core Philosophy & Constraints
2.1 The "Zero-Trust" Network Assumption

We assume the internet, the phone line, and the video feed are compromised.

    Offline by Design: The app has no internet permissions. It cannot "leak" keys because it cannot reach a server.

    Ephemeral Existence: Keys are generated on-device and never leave the secure enclave except for the millisecond they are displayed to the user's eye.

2.2 The "Index-First" Visibility Protocol

To prevent coercion and "look-ahead" attacks, we utilize a strictly ordered verification flow:

    Coordinate then Reveal: Users must verbally agree on a visible "Page Number" (Index) before unlocking the secret word.

    Burn-on-Sight: Once a secret word is viewed, it is cryptographically erased from storage. It cannot be recovered.

    Forward Only: You can skip forward to catch up to a peer, but you can never look back.

2.3 Fail-Secure Revocation

If a contact is suspected of being compromised, the system favors security over convenience. Revocation immediately wipes local keys, forcing a new physical meeting or secure setup to re-establish trust.
3. Security Architecture
3.1 The "Vault" (Storage)

    Encryption: Database is encrypted via SQLCipher (AES-256-GCM).

    Key Wrapping: The database key is wrapped by the device hardware (Secure Enclave / StrongBox).

    Biometric Gate: Unlocking the app requires FaceID/Fingerprint every time.

3.2 Anti-Forensics

    Root/Jailbreak Detection: The app will refuse to launch if the OS integrity is compromised.

    Screen Shield: The app view is obscured (blurred) in the OS "Recent Apps" switcher.

    Panic Shred: Entering a specific "Duress PIN" instantly deletes the database key, rendering the data mathematically irretrievable.

4. Data Models
4.1 Peer Identity (Peer)

The trusted contact entity.
JSON

{
  "uuid": "UUIDv4",
  "displayName": "Alice (CFO)",
  "publicKey": "Ed25519_Base64",
  "currentLocalIndex": 12,    // The next key *I* will use
  "pairingMethod": "ENUM (NFC, QR_SHADOW_DROP)",
  "created_at": "Timestamp"
}

4.2 The One-Time Pad (SessionBundle)

The "fuel" for verification. Derived from a single Seed to allow efficient storage and QR transfer.
JSON

{
  "sessionId": "Random_128bit_Hex",
  "peerId": "Link -> Peer.uuid",
  "rootSeed": "Encrypted_32Bytes", // Only decrypted to derive words
  "burnedIndexes": [0, 1, 2... 11] // Bitmask of consumed keys
}

5. Protocols & Algorithms
5.1 Key Creation A: The NFC Handshake (Physical)

Best for: In-person meetings.

    Initiation: Users A and B enter "Handshake Mode."

    Exchange: Devices touch. An Ephemeral Elliptic Curve Diffie-Hellman (ECDH) exchange occurs over NFC.

    Derivation: Both devices derive a Shared Secret (S).

    Verification: A "Short Authentication String" (SAS) is shown on both screens to confirm no Man-in-the-Middle.

    Storage: The list is initialized, and S is wiped from memory.

5.2 Key Creation B: The "Shadow-Drop" QR (Remote)

Best for: Remote setup over video calls.

This protocol solves the "Chicken and Egg" problem of sharing a secret key over a compromised video channel without the camera capturing the raw secret.

    Encrypt: Alice enters a verbal PIN (e.g., 5521). The App encrypts the Seed with this PIN (KeyPIN​).
    Payload=AES256(Seed,KeyPIN​)

    Display: Alice's screen shows a QR code containing [Salt + IV + Payload].

    Scan: Bob scans the QR. His app detects the header and locks the payload.

    Decrypt: Bob asks Alice for the PIN verbally. Alice says "5-5-2-1".

    Derivation: Bob enters the PIN. Both sides now possess the same Seed without it ever being exposed "naked" to the camera or screenshot malware.

5.3 Verification: The "Index-First" Protocol

Goal: Verify identity without leaking secrets or getting out of sync.

    Announce: Alice says, "I am on Key #14."

    Align: Bob looks at his dial.

        Case A (Sync): He is on #14. He replies "Confirmed."

        Case B (Lag): He is on #10. He swipes up 4 times (burning 10-13) to reach #14. He replies "Okay, caught up. Ready."

    Challenge: Alice reveals her word: "STAIRWAY."

    Response: Bob reveals his word.

        Match: Identity Verified. The session advances to #15.

        Mismatch: SECURITY ALERT. Hang up immediately.

6. UI/UX Design System
6.1 Visual Aesthetics

    Theme: "Dark Mode First" (Cyber-Security aesthetic).

    Typography: JetBrains Mono for all codes/words. Inter for UI labels.

    Palette: Deep Slate (#0F172A), Pale Silver (#E2E8F0), Neon Green (#00E676), Crimson Red (#FF1744).

6.2 The "Verification Dial" (Main Screen)

Instead of a static card, the UI uses a Vertical Rolodex metaphor.

    Center Focus: The current active Index (e.g., "14") is large and center.

    The Swipe: User swipes UP to discard a key (Skip) and move to the next index.

    The Reveal: Long-Pressing the center number "unlocks" it, revealing the word (e.g., "BATTERY") and locking the choice.

    Feedback: The background pulses Green on reveal.

7. Technology Stack

    Framework: Flutter (Dart) - Cross-platform, high performance.

    Database: Isar (NoSQL) + SQLCipher Encryption.

    Secure Storage: flutter_secure_storage (Keychain/Keystore) for Master Key wrapping.

    Cryptography: cryptography (Dart FFI).

        Algorithm: XChaCha20-Poly1305 for encryption. Argon2id for PIN hashing.

    Hardware: nfc_manager (NFC), mobile_scanner (QR), local_auth (Biometrics).

8. Future Roadmap

    Post-Quantum Cryptography: Migrating from ECDH to Kyber/Dilithium algorithms once mobile libraries mature.

    Wearable Integration: Viewing the verification word directly on a paired Smart Watch for discreet checking during in-person meetings.

    Enterprise MDM: Allow companies to pre-seed keys via secure USB provisioning for employees.
