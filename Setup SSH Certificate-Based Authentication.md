<section class="link-embed" data-url="https://blog.nashcom.de/nashcomblog.nsf/dx/using-an-ssh-certificate-authority-ca-for-centralized-ssh-access.htm" data-embed-type="opengraph" data-title="Daniel Nashed's Blog" data-description="Using an SSH Certificate Authority (CA) for Centralized SSH Access" data-favicon="api/attachments/WhDP7raSU4jJ/image/blog.nashcom.de.ico"><a href="https://blog.nashcom.de/nashcomblog.nsf/dx/using-an-ssh-certificate-authority-ca-for-centralized-ssh-access.htm">Daniel Nashed's Blog</a></section>

#### **1\. Generate two SSH CA Key Pairs**

On a secure administrative machine (your CA host), generate an SSH key pair to act as your SSH CA:   

```
ssh-keygen -t ed25519 -f ~/.ssh/ssh_user_ca -C "SSH User Certificate Authority"
ssh-keygen -t ed25519 -f ~/.ssh/ssh_host_ca -C "SSH Host Certificate Authority"
```

This creates:

*   `ssh_user_ca` (CA private key — guard this like a root key)
*   `ssh_user_ca.pub` (CA public key — distributed to servers)
*   `ssh_host_ca` (CA private key — guard this like a root key)
*   `ssh_host_ca.pub` (CA public key — distributed to servers)

#### **2\. Configure the SSH Server to Trust the CA**

Copy `ssh_user_ca.pub` to the target server (e.g., `/etc/ssh/ssh_user_ca.pub`).   

Edit `/etc/ssh/sshd_config` on the server: 

```
# Trust any user certificate signed by this CA
TrustedUserCAKeys /etc/ssh/ssh_user_ca.pub
​
# Optional: Map principals to local system users
AuthorizedPrincipalsFile /etc/ssh/auth_principals/%u
```

Restart the SSH daemon:

```
sudo systemctl restart sshd
```

_(Optional: If using_ `_AuthorizedPrincipalsFile_`_, create_ `_/etc/ssh/auth_principals/ubuntu_` _containing allowed principals such as_ `_dev-admin_` _or_ `_developers_`_.)_

#### **3\. Sign a User's SSH Key**

When a user generates their standard SSH key (`ssh-keygen -t ed25519 -f srv_id_ed25519`), take their public key (`srv_id_ed25519.pub`) and sign it using the CA private key: 

```
ssh-keygen -s ~/.ssh/ssh_user_ca \
  -I "johndoe@example" \
  -n "ubuntu,breakglass" \
  -V +52w \
  -z 1001 \
  srv_id_ed25519.pub
```

*   `-s`: The CA private key used for signing.
*   `-I`: A human-readable identity string (logged in `auth.log`).
*   `-n`: Allowed **principals** (usernames/roles this cert is allowed to log in as).
*   `-V`: Validity period (e.g., `+52w` for 52 weeks, `+8h` for 8 hours, or a specific date range).
*   `-z`: Optional serial number.

This generates `srv_id_ed25519-cert.pub` in the same directory as the public key.

#### **4\. Connect with the Certificate**

Place `srv_id_ed25519` and `srv_id_ed25519-cert.pub` side-by-side in `~/.ssh/` on the client machine. OpenSSH automatically loads the certificate when using the private key.

```
Host LINUX_HOST
  HostName                  IP_ADDRESS_OF_HOST
  Port                      22
  User                      ubuntu
  IdentityFile              ~/.ssh/srv_id_ed25519
  CertificateFile           ~/.ssh/srv_id_ed25519-cert.pub
  ServerAliveInterval       5
  ExitOnForwardFailure      yes
```

To view the certificate details and validity period:

```
ssh-keygen -L -f ~/.ssh/srv_id_ed25519-cert.pub
```

---

This guide assumes user authentication already works with `ssh LINUX_HOST`, and you have a host CA key pair on your local signing machine:

```
~/.ssh/ssh_host_ca       # CA private key
~/.ssh/ssh_host_ca.pub   # CA public key
```

Replace `LINUX_HOST` throughout with your actual SSH alias and desired host identity, preferably lowercase. Run local commands in the same terminal so the temporary-directory variable remains available.

**1\. Create a temporary working directory locally**

```
host_cert_dir=$(mktemp -d "${TMPDIR:-/tmp}/ssh-host-cert.XXXXXXXX")
```

This directory will hold the server’s public key and the signed certificate during deployment.

**2\. Retrieve the server’s existing public host key**

On your local machine:

```
ssh -o StrictHostKeyChecking=yes LINUX_HOST \
  'cat /etc/ssh/ssh_host_ed25519_key.pub' \
  > "$host_cert_dir/ssh_host_ed25519_key.pub"

ssh-keygen -lf "$host_cert_dir/ssh_host_ed25519_key.pub"
```

This requires an already trusted host-key entry. If the host is unknown, verify its fingerprint through a trusted channel before proceeding.

The server’s private host key remains on the server.

**3\. Sign the public key locally**

Choose an unused serial number for your CA:

```
host_cert_serial=1001

ssh-keygen \
  -s ~/.ssh/ssh_host_ca \
  -I "LINUX_HOST-host-$host_cert_serial" \
  -h \
  -n LINUX_HOST \
  -V -5m:+90d \
  -z "$host_cert_serial" \
  "$host_cert_dir/ssh_host_ed25519_key.pub"
```

The options mean:

| Option | Purpose |
| --- | --- |
| `-s` | Select the CA signing key |
| `-I` | Set an administrative certificate identifier |
| `-h` | Issue a host certificate |
| `-n` | Specify the authorized host identity |
| `-V` | Allow five minutes of clock skew and expire after 90 days |
| `-z` | Set the certificate serial |

Inspect the result:

```
ssh-keygen -L \
  -f "$host_cert_dir/ssh_host_ed25519_key-cert.pub"

ssh-keygen -lf ~/.ssh/ssh_host_ca.pub
```

Confirm the certificate type is **host**, its principal is `LINUX_HOST`, and its signing CA fingerprint matches your CA.

**4\. Transfer the certificate to the server**

On your local machine:

```
scp "$host_cert_dir/ssh_host_ed25519_key-cert.pub" \
  LINUX_HOST:~/ssh_host_ed25519_key-cert.pub

ssh LINUX_HOST
```

Keep this server session open during installation and testing.

Only the public certificate is transferred. The CA private key stays on the signing machine.

**5\. Install and enable the certificate on the server**

On `LINUX_HOST`:

```
sudo install -o root -g root -m 0644 \
  ~/ssh_host_ed25519_key-cert.pub \
  /etc/ssh/ssh_host_ed25519_key-cert.pub
```

If `/etc/ssh/sshd_config` includes `/etc/ssh/sshd_config.d/*.conf`, create a configuration snippet:

```
printf '%s\n' \
  'HostCertificate /etc/ssh/ssh_host_ed25519_key-cert.pub' \
  | sudo tee /etc/ssh/sshd_config.d/20-host-certificate.conf
```

Otherwise, add that `HostCertificate` line to the global section of `/etc/ssh/sshd_config`, before any `Match` blocks.

The server must already load the matching private key:

```
HostKey /etc/ssh/ssh_host_ed25519_key
```

Validate the configuration:

```
sudo /usr/sbin/sshd -t
```

If validation succeeds, reload the service:

```
# Debian / Ubuntu
sudo systemctl reload ssh

# Arch / Fedora / RHEL: use this instead
sudo systemctl reload sshd
```

Remove the uploaded staging copy:

```
rm ~/ssh_host_ed25519_key-cert.pub
```

**6\. Configure CA trust on each client**

Each client needs an authentic copy of `ssh_host_ca.pub`. Verify its fingerprint against the signing machine before trusting it.

On each client, run once:

```
mkdir -p ~/.ssh
chmod 700 ~/.ssh

awk '{print "@cert-authority LINUX_HOST " $1 " " $2}' \
  ~/.ssh/ssh_host_ca.pub >> ~/.ssh/known_hosts

chmod 600 ~/.ssh/known_hosts
```

This authorizes your CA to certify the identity `LINUX_HOST`.

Add these lines inside the existing `Host LINUX_HOST` block in `~/.ssh/config`:

```
Host LINUX_HOST
  # Retain your existing HostName, User, and user authentication settings.
  HostKeyAlias LINUX_HOST
  StrictHostKeyChecking yes
  HostKeyAlgorithms ssh-ed25519-cert-v01@openssh.com
```

`HostKeyAlias` selects the identity to validate. The algorithm setting requires an Ed25519 host certificate.

**7\. Test from a second terminal**

On a configured client:

```
ssh -v LINUX_HOST true
```

Confirm the log contains:

```
Host 'LINUX_HOST' is known and matches the ED25519-CERT host certificate.
Found CA key in ...known_hosts...
```

The command should finish with:

```
Exit status 0
```

You can then close the deployment session.

**8\. Remove the local working files**

In the original local terminal:

```
rm -- \
  "$host_cert_dir/ssh_host_ed25519_key.pub" \
  "$host_cert_dir/ssh_host_ed25519_key-cert.pub"

rmdir -- "$host_cert_dir"
unset host_cert_dir host_cert_serial
```

The local deployment copies are no longer needed. Keep these files on the **server**:

```
/etc/ssh/ssh_host_ed25519_key
/etc/ssh/ssh_host_ed25519_key.pub
/etc/ssh/ssh_host_ed25519_key-cert.pub
```

For renewal, retrieve the host public key again, sign it with a new serial and validity period, install the replacement certificate, and reload SSH. Clients continue trusting the same CA without changes.

---

Create this recovery file while normal certificate-based access still works.

**1\. Retrieve the host public key into a temporary folder**

On your local machine:

```
host_recovery_dir=$(mktemp -d)

ssh -o StrictHostKeyChecking=yes LINUX_HOST \
  'cat /etc/ssh/ssh_host_ed25519_key.pub' \
  > "$host_recovery_dir/host_key.pub"
```

**2\. Create the recovery trust file**

Use `LINUX_HOST` as the recovery identity:

```
awk '{print "LINUX_HOST " $1 " " $2}' \
  "$host_recovery_dir/host_key.pub" \
  > ~/.ssh/known_hosts_LINUX_HOST_recovery

chmod 600 ~/.ssh/known_hosts_LINUX_HOST_recovery
```

This stores the server’s raw public host key as a trusted pin.

**3\. Test recovery access**

```
ssh -v \
  -o HostKeyAlias=LINUX_HOST \
  -o HostKeyAlgorithms=ssh-ed25519 \
  -o UserKnownHostsFile=~/.ssh/known_hosts_LINUX_HOST_recovery \
  -o StrictHostKeyChecking=yes \
  LINUX_HOST true
```

Look for a matching `ED25519 host key` and `Exit status 0`.

This connection can work after the **host certificate expires**, provided the server’s host key stays unchanged and your user credentials remain valid. Remove the trailing `true` to open a shell.

**4\. Delete the temporary public-key copy**

```
rm -- "$host_recovery_dir/host_key.pub"
rmdir -- "$host_recovery_dir"
unset host_recovery_dir
```

Keep `~/.ssh/known_hosts_LINUX_HOST_recovery`; it contains the public-key pin needed for recovery.