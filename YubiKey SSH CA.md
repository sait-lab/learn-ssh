# Redundant Hardware-Backed SSH Certificate Authority on macOS

This document provides a field-tested, step-by-step operational guide for generating an offline SSH Certificate Authority (CA) private key in an ephemeral RAM disk and importing it into multiple redundant YubiKeys for high availability and disaster recovery.

---

## 1. Architecture & Design Rationale

```
                     +-------------------------------+
                     | Generate CA Key in RAM Disk   |
                     |     (/Volumes/RAM-Disk)       |
                     +---------------+---------------+
                                     |
             +-----------------------+-----------------------+
             |                                               |
             v                                               v
     +---------------+                               +---------------+
     |  YubiKey 1    |                               |  YubiKey 2    |
     | (Primary CA)  |                               |  (Backup CA)  |
     | S/N: 25848218 |                               | S/N: 25848219 |
     | Slot 9c (P384)|                               | Slot 9c (P384)|
     +---------------+                               +---------------+
```

### Why Offline Generation?
Private keys generated directly inside a YubiKey (`ykman piv keys generate`) can never be read or exported by design. If a single key is lost or fails, the CA is permanently destroyed, requiring re-issuance of all client certificates and server configurations. Generating the key offline in volatile memory allows importing identical cryptographic material into two or more physical tokens before wiping memory.

### Algorithm Selection: ECDSA P-384 vs Ed25519
* **YubiKey Firmware 5.7.0+** supports **`Ed25519`**.
* **YubiKey Firmware ≤ 5.4.x** (such as F/W 5.4.3) supports **`RSA2048`**, **`ECCP256`**, and **`ECCP384`**.
* **Decision:** We use **NIST P-384 (`ECCP384` / `ecdsa-sha2-nistp384`)**, the strongest elliptic curve supported on firmware 5.4.x, offering military-grade security and broad OpenSSH compatibility.

---

## 2. Prerequisites & Software Installation

On macOS, install the required command-line tools via Homebrew:

```bash
# YubiKey Manager CLI
brew install ykman
# Or add alias in shell rc
alias ykman="uvx --from yubikey-manager ykman "

# Modern OpenSSL CLI (OpenSSL 4)
brew install openssl@4

# OpenSSH PKCS#11 module (provides libykcs11.dylib)
brew install yubico-piv-tool
```

Confirm tool versions:
```bash
ykman --version
# YubiKey Manager (ykman) version: 5.9.2

openssl version
# OpenSSL 4.1.0 23 Oct 2026 (Library: OpenSSL 4.1.0 23 Oct 2026)
```

---

## 3. Step-by-Step Procedure

### Step 3.1: Mount & Prepare the Ephemeral RAM Disk

To guarantee that private key material is never written to non-volatile physical storage (NVMe/SSD wear-leveling blocks), perform all generation in an ephemeral RAM disk:

```bash
# If not already mounted, create a 64MB RAM disk:
# RAM_DEV=$(hdiutil attach -nomount ram://131072)
# newfs_hfs -v "RAM-Disk" $RAM_DEV

mkdir -p /Volumes/RAM-Disk/ca-setup
chmod 700 /Volumes/RAM-Disk/ca-setup
cd /Volumes/RAM-Disk/ca-setup
```

---

### Step 3.2: Generate the CA Private Key and X.509 Certificate

PIV slot `9c` requires both the private key and an accompanying X.509 certificate container:

```bash
# 1. Generate the ECDSA P-384 private key
openssl ecparam -name secp384r1 -genkey -noout -out ca_key.pem

# 2. Generate a self-signed X.509 certificate (valid for 20 years)
openssl req -new -x509 -key ca_key.pem -out ca_cert.pem -days 7305 \
  -subj "/CN=SSH Certificate Authority/"
```

---

### Step 3.3: Import into Primary YubiKey (Key 1)

1. Insert your **first YubiKey** (e.g., S/N `25848218`).
2. Import the private key and certificate into slot `9c` (Digital Signature):

```bash
# Import private key with security policies
ykman piv keys import 9c ca_key.pem \
  --pin-policy ONCE \
  --touch-policy ALWAYS

# Import the matching X.509 certificate
ykman piv certificates import 9c ca_cert.pem
```

> **Security Policies Applied:**
> * `--touch-policy ALWAYS`: Physical touch required for every single signature. Malware running in the background cannot sign certificates without user interaction.
> * `--pin-policy ONCE`: Requires the PIV PIN once per session.

---

### Step 3.4: Import into Secondary (Backup) YubiKey (Key 2)

1. Unplug Key 1.
2. Insert your **second YubiKey** (e.g., S/N `25848219`).
3. Run the exact same import commands:

```bash
ykman piv keys import 9c ca_key.pem \
  --pin-policy ONCE \
  --touch-policy ALWAYS

ykman piv certificates import 9c ca_cert.pem
```

Both tokens now contain identical cryptographic keys and certificates in slot `9c`.

---

### Step 3.5: Export & Isolate the CA Public Key

1. Keep either YubiKey inserted.
2. Identify the PKCS#11 library path:
   * **Apple Silicon:** `/opt/homebrew/lib/libykcs11.dylib`
   * **Intel Mac:** `/usr/local/lib/libykcs11.dylib`
   * Automated: `YKCS11_LIB="$(brew --prefix)/lib/libykcs11.dylib"`

3. Export the smartcard public keys:
   ```bash
   YKCS11_LIB="$(brew --prefix)/lib/libykcs11.dylib"
   ssh-keygen -D "$YKCS11_LIB" -e > ~/.ssh/ssh_yubikey_ca_all.pub
   ```

4. **Isolate the Digital Signature Key:**
   `ssh-keygen -D` outputs all card slots, including Yubico's factory attestation key. Filter the output to retain only the `Digital Signature` line:
   ```bash
   grep "Digital Signature" ~/.ssh/ssh_yubikey_ca_all.pub > ~/.ssh/ssh_yubikey_ca.pub
   rm -f ~/.ssh/ssh_yubikey_ca_all.pub
   ```

5. Confirm the exported public key:
   ```bash
   cat ~/.ssh/ssh_yubikey_ca.pub
   ```
   *Expected Output Format:*
   ```text
   ecdsa-sha2-nistp384 AAAAE2VjZHNhLXNoYTItbmlzdHAzODQAAAAIbmlzdHAzODQAAABhBIomKK... Public key for Digital Signature
   ```

---

### Step 3.6: Wipe Volatile Artifacts and Unmount RAM Disk

Now that both physical keys are provisioned, permanently purge all in-memory plain text keys:

```bash
# Multi-pass secure wipe of the PEM files
rm -P /Volumes/RAM-Disk/ca-setup/ca_key.pem /Volumes/RAM-Disk/ca-setup/ca_cert.pem

# Delete working directory
cd ~
rm -rf /Volumes/RAM-Disk/ca-setup

# Eject RAM disk to free memory pages
diskutil eject /Volumes/RAM-Disk
```

---

## 4. Verification & Testing

Verify that both physical keys produce identical signatures and CA fingerprints.

### Test Key 1 (Primary):
1. Insert Key 1.
2. Generate a test user key and sign it:
   ```bash
   cd ~/.ssh
   
   ssh-keygen -t ed25519 -f ./id_test -N ""
   
   ssh-keygen -D "$YKCS11_LIB" \
     -s ~/.ssh/ssh_yubikey_ca.pub \
     -I "test-alice" \
     -n "alice,admin" \
     -V +1d \
     ./id_test.pub
   ```
3. Inspect the resulting certificate:
   ```bash
   ssh-keygen -L -f ./id_test-cert.pub
   ```
   Note the `Signing CA` fingerprint:
   `Signing CA: ECDSA SHA256:9hoNLvvemBhzkpHsiY6fFcMiVzQ0khVtv08H07isgmk`

### Test Key 2 (Secondary / Backup):
1. Swap in Key 2.
2. Sign a second test certificate:
   ```bash
   ssh-keygen -D "$YKCS11_LIB" \
     -s ~/.ssh/ssh_yubikey_ca.pub \
     -I "test-bob" \
     -n "bob" \
     -V +1d \
     ./id_test.pub
   ```
3. Inspect the certificate:
   ```bash
   ssh-keygen -L -f ./id_test-cert.pub
   ```
   Verify that the `Signing CA` SHA256 fingerprint matches identically:
   `Signing CA: ECDSA SHA256:9hoNLvvemBhzkpHsiY6fFcMiVzQ0khVtv08H07isgmk`

Clean up test keys:
```bash
rm -f ./id_test ./id_test.pub ./id_test-cert.pub
```

---

## 5. Server-Side Deployment

To configure target hosts to trust user certificates issued by either YubiKey:

1. **Copy the Public CA Key:**
   Deploy `~/.ssh/ssh_yubikey_ca.pub` to the target server at `/etc/ssh/ssh_ca.pub`:
   
   ```bash
   sudo cp ssh_yubikey_ca.pub /etc/ssh/ssh_ca.pub
   sudo chmod 644 /etc/ssh/ssh_ca.pub
   ```
   
2. **Configure OpenSSH Server (`/etc/ssh/sshd_config`):**
   ```sshd_config
   TrustedUserCAKeys /etc/ssh/ssh_ca.pub
   ```

3. **Reload `sshd`:**
   ```bash
   sudo systemctl reload sshd   # Linux systemd
   # or
   sudo launchctl kickstart -k system/com.openssh.sshd  # macOS
   ```

> **Client Logins Require Zero CA Hardware Interaction:**
> When users authenticate, the server mathematically verifies the certificate signature using the public `/etc/ssh/ssh_ca.pub`. The YubiKeys do not need to be connected during client SSH sessions.

If the server already has a `/etc/ssh/ssh_ca.pub` file, append the key

Run this from the client to append the key directly to the server:

```
cat ~/.ssh/ssh_yubikey_ca.pub | ssh user@your-server "sudo tee -a /etc/ssh/ssh_user_ca.pub"
```

Or on the server directly

```
sudo bash -c 'cat ssh_yubikey_ca.pub >> /etc/ssh/ssh_user_ca.pub'
```

Check the contents of `/etc/ssh/ssh_user_ca.pub` on the server:

```
cat /etc/ssh/ssh_user_ca.pub
```

It should look like this (multiple distinct lines):

```
ssh-ed25519 AAAAC3... your-original-ca-key
ecdsa-sha2-nistp384 AAAAE2Vj... Public key for Digital Signature
```

Reload the SSH daemon to apply the change (this will not drop your active connection):

```
sudo systemctl reload sshd
```

---

## 6. Sign a User's SSH Key

When a user generates their standard SSH key (`ssh-keygen -t ed25519 -f ssh_user_id_ed25519`), take their public key (`srv_id_ed25519.pub`) and sign it using the CA private key:

```
YKCS11_LIB="$(brew --prefix)/lib/libykcs11.dylib"

ssh-keygen -D "$YKCS11_LIB" \
  -s ~/.ssh/ssh_yubikey_ca.pub \
  -I "johndoe@example.com" \
  -n "ubuntu,breakglass" \
  -V +52w \
  -z 1001 \
  ~/.ssh/ssh_user_id_ed25519.pub
```

Inspect the resulting certificate:

```
ssh-keygen -L -f ~/.ssh/ssh_user_id_ed25519-cert.pub
~/.ssh/ssh_user_id_ed25519-cert.pub:
        Type: ssh-ed25519-cert-v01@openssh.com user certificate
        Public key: ED25519-CERT SHA256:q2z0soZnaMgKrQZYC3S5DHVF8L5MwpOvv22V/1SH5jE
        Signing CA: ECDSA SHA256:9hoNLvvemBhzkpHsiY6fFcMiVzQ0khVtv08H07isgmk (using ecdsa-sha2-nistp384)
        Key ID: "johndoe@example.com"
        Serial: 1001
        Valid: from 2026-11-01T16:36:00 to 2027-10-31T16:37:36
        Principals:
                ubuntu
                breakglass
        Critical Options: (none)
        Extensions:
                permit-X11-forwarding
                permit-agent-forwarding
                permit-port-forwarding
                permit-pty
```

