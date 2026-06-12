# The 4-Way Handshake in Wi-Fi (WPA/WPA2)

## Overview

The **4-way handshake** is a critical process in **WPA (Wi-Fi Protected Access)** and **WPA2** networks that establishes a secure connection between a client (e.g., laptop, smartphone) and an access point (AP). It ensures that both devices have the correct encryption keys and can communicate securely.

## Why is the 4-Way Handshake Important?

1. **Mutual Authentication** – Confirms that both the client and AP know the Pre-Shared Key (PSK)
2. **Key Exchange** – Derives unique encryption keys for the session
3. **Prevents Replay Attacks** – Uses random numbers (nonces) to ensure freshness
4. **Establishes Trust** – Both parties verify each other before exchanging data

## How the 4-Way Handshake Works

### Overview Diagram

```
┌──────────┐                              ┌──────┐
│  Client  │                              │  AP  │
│(Supplicant)                    (Authenticator)
└────┬─────┘                              └──┬───┘
     │                                       │
     │   1. ANonce (Random Number)           │
     │<──────────────────────────────────────┤
     │       (AP sends first)                │
     │                                       │
     │   2. SNonce + MIC                     │
     ├──────────────────────────────────────>│
     │       (Client responds)               │
     │                                       │
     │   3. GTK + MIC + Confirmation         │
     │<──────────────────────────────────────┤
     │       (AP sends group key)            │
     │                                       │
     │   4. ACK (Acknowledgment)             │
     ├──────────────────────────────────────>│
     │       (Client confirms)               │
     │                                       │
     │  ✓ Secure Connection Established      │
```

## Detailed Step-by-Step Process

### Message 1: AP → Client (ANonce)

**What Happens:**
- The AP generates a random number: **ANonce (Authenticator Nonce)**
- This random number is sent to the client
- The client uses ANonce + PSK to start deriving the encryption key

**Client Side Processing:**
```
PSK (Password) + SSID + ANonce + SNonce (will generate) + MAC Addresses
                ↓
        PMK (Pairwise Master Key) generated
                ↓
        PTK (Pairwise Transient Key) partially computed
```

**Frame Details:**
- Frame Type: EAPOL (Extensible Authentication Protocol Over LAN)
- Nonce: ANonce (random 32-byte value)
- Replay Counter: 0 (first message)
- Key Information: Flags set

---

### Message 2: Client → AP (SNonce + MIC)

**What Happens:**
- The client generates its own random number: **SNonce (Supplicant Nonce)**
- The client computes the **MIC (Message Integrity Code)** using the PTK (partially derived)
- Both ANonce and SNonce are sent to derive the final PTK

**Client Side Calculations:**
```
Client generates SNonce (random)
         ↓
Computes PTK using:
  - PSK (shared password)
  - SSID
  - ANonce (from AP)
  - SNonce (just generated)
  - Client MAC address
  - AP MAC address
         ↓
Computes MIC using PTK to verify message integrity
         ↓
Sends SNonce + MIC to AP
```

**Message Format:**
```
┌─────────────────────────────────────┐
│  EAPOL Key Frame                    │
├─────────────────────────────────────┤
│ Frame Type: Key                     │
│ Replay Counter: 1                   │
│ Key Nonce: SNonce                   │
│ RSSC: 0                             │
│ Key RSC: 0                          │
│ Key MIC: [MIC Value]                │
│ Key Data Length: 0                  │
│ Key Data: (None)                    │
└─────────────────────────────────────┘
```

**AP Side Calculations:**
```
AP now has ANonce + SNonce + PSK + MAC Addresses
         ↓
Computes the same PTK
         ↓
Verifies MIC using computed PTK
         ↓
If MIC matches: Client knows the PSK ✓
         ↓
AP proceeds to Message 3
```

---

### Message 3: AP → Client (GTK + MIC)

**What Happens:**
- The AP generates the **GTK (Group Temporal Key)** for multicast/broadcast traffic
- AP sends GTK encrypted with the PTK
- Includes a new MIC for verification

**Key Components:**
```
PTK Components:
├─ KCK (Key Confirmation Key) - for MIC
├─ KEK (Key Encryption Key) - encrypts data
├─ TK (Temporal Key) - for unicast encryption

GTK:
├─ Used for multicast/broadcast traffic
├─ Group Key Index
├─ Encrypted with KEK
```

**Message Format:**
```
┌─────────────────────────────────────┐
│  EAPOL Key Frame (Message 3)        │
├─────────────────────────────────────┤
│ Frame Type: Key                     │
│ Key Information:                    │
│   - Key Type: Group (1) / Unicast   │
│   - MIC: 1                          │
│   - Secure: 1                       │
│   - Error: 0                        │
│   - Request: 0                      │
│ Replay Counter: 2                   │
│ Key Nonce: ANonce                   │
│ Key RSNX: GTK                       │
│ Key RSC: Sequence counter           │
│ Key ID: Group Key Index             │
│ Key MIC: [MIC of entire frame]      │
│ Key Data Length: [encrypted GTK]    │
│ Key Data: [Encrypted GTK]           │
└─────────────────────────────────────┘
```

**AP Sends:**
- Own ANonce
- GTK (encrypted with KEK)
- MIC for verification
- Replay counter incremented

**Client Side Processing:**
```
Receives Message 3
         ↓
Verifies MIC using KCK
         ↓
If MIC valid: AP is authentic ✓
         ↓
Decrypts GTK using KEK
         ↓
Installs PTK + GTK for encryption
```

---

### Message 4: Client → AP (ACK)

**What Happens:**
- Client confirms receipt of Message 3
- Sends an acknowledgment frame
- Completes the handshake
- Both sides now have identical keys

**Message Format:**
```
┌─────────────────────────────────────┐
│  EAPOL Key Frame (Message 4)        │
├─────────────────────────────────────┤
│ Frame Type: Key                     │
│ Key Information:                    │
│   - Key Type: Group / Unicast       │
│   - MIC: 1                          │
│   - Secure: 1                       │
│   - Install: 0 (already installed)  │
│ Replay Counter: 3                   │
│ Key Nonce: 0                        │
│ Key RSC: 0                          │
│ Key MIC: [MIC of frame]             │
│ Key Data Length: 0 (No data)        │
└─────────────────────────────────────┘
```

**Final Result:**
```
┌─────────────────────────────────────┐
│  Connection Established             │
├─────────────────────────────────────┤
│ Client & AP share:                  │
│ ✓ PTK (Unicast encryption)          │
│ ✓ GTK (Multicast encryption)        │
│ ✓ Same nonces verified              │
│ ✓ Mutual authentication complete    │
│ ✓ All data now encrypted            │
└─────────────────────────────────────┘
```

## Key Components Explained

### Nonces (Random Numbers)

**ANonce (Authenticator Nonce)**
- Generated by AP
- 256-bit random value
- Used in key derivation
- Changes for each connection

**SNonce (Supplicant Nonce)**
- Generated by client
- 256-bit random value
- Used in key derivation
- Proves client knows PSK

### Keys Generated

**PMK (Pairwise Master Key)**
```
PMK = PBKDF2-SHA1(PSK, SSID, 4096 iterations, 256 bits)
- Derived from password
- Same for all devices on same SSID
- Stored on AP for faster reconnection
```

**PTK (Pairwise Transient Key)**
```
PTK = PRF-SHA256(PMK || "Pairwise key expansion" || 
                  MIN(MAC_A, MAC_B) || MAX(MAC_A, MAC_B) ||
                  MIN(ANonce, SNonce) || MAX(ANonce, SNonce))
                  
- 384 bits total:
  ├─ KCK (128 bits) - Key Confirmation Key (MIC)
  ├─ KEK (128 bits) - Key Encryption Key
  └─ TK (128 bits) - Temporal Key (actual encryption)
```

**GTK (Group Transient Key)**
```
GTK = Random 256-bit value generated by AP
- For multicast/broadcast traffic
- Same for all clients on AP
- Sent encrypted in Message 3
- Periodically refreshed
```

### MIC (Message Integrity Code)

**Purpose:**
- Ensures message wasn't modified
- Authenticates sender
- Computed using KCK (part of PTK)

**Computation:**
```
MIC = HMAC-SHA1 or HMAC-SHA256(KCK, EAPOL_Frame)
```

**Verification:**
- Receiver computes MIC
- Compares with received MIC
- If different: attack detected, frame rejected

## Why the 4-Way Handshake Can Be Attacked

### Vulnerabilities

**1. KRACK Attack (Key Reinstallation Attack)**
- **Discovered**: 2017 by Mathy Vanhoef and Frank Piessens
- **How it works**:
  - Attacker replays Message 3
  - Forces key reinstallation
  - Can reset nonce/encryption state
  - Allows decryption of traffic

- **Prevention**:
  - Update firmware/OS
  - WPA3 resistant

**2. Offline Brute-Force Attack**
- **Process**:
  ```
  1. Capture complete 4-way handshake
  2. Try passwords from wordlist
  3. Compute PTK for each password
  4. Verify MIC against captured MIC
  5. If match: password found ✓
  ```
- **Time**: Depends on wordlist size and hardware
  - Fast wordlist: Minutes
  - Large wordlist: Hours to days

**3. Weak Passwords**
- Only 8-character passwords
- Dictionary words
- No special characters
- Easily cracked

**4. PMKID Attack**
- Capture PMKID from single frame
- No handshake needed
- Faster offline attack
- Requires single beacon frame

## Wireshark Analysis

### Capturing Handshake in Wireshark

```
Filter: eapol

Observe:
1. Message 1: Nonce from AP (Key Information bit 7 = 0)
2. Message 2: Nonce from client (Key Information bit 7 = 1)
3. Message 3: GTK from AP (MIC present)
4. Message 4: Client ACK (MIC present)
```

### Packet Structure

```
Frame
├─ Ethernet Header
├─ IP Header (if logged)
├─ LLC Header
└─ EAPOL
   ├─ EAPOL Header
   ├─ Key Descriptor Type
   └─ Key Data Payload
```

## Protection Mechanisms

### In WPA2

- **Authentication**: PSK verified through MIC
- **Confidentiality**: PTK for encryption
- **Integrity**: MIC protects all messages
- **Freshness**: Nonces prevent replay

### In WPA3

- **SAE (Simultaneous Authentication of Equals)**
  - Replaces PSK exchange
  - Protected against offline attacks
  - Stronger math-based authentication

- **Individualized Data Encryption (OWE)**
  - Works without password
  - Still establishes unique keys

---

**Next**: Learn about practical [Aircrack-ng Guide](../06-Aircrack-ng-Guide/README.md)
