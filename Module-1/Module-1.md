# Module 1: Secure Server Foundation

**Goal:** Build a secure, maintainable Ubuntu server environment before deploying applications or code.

## Learning Objectives

By the end of this module, you will be able to:

- Provision an Ubuntu server on DigitalOcean.
- Generate and configure SSH keys using Ed25519 or RSA.
- Understand public-key authentication and the SSH security model.
- Create a non-root administrative user and restrict root access.
- Disable SSH password authentication.
- Configure SSH client shortcuts for efficient administration.
- Apply firewall rules using UFW.
- Configure Fail2Ban to mitigate repeated SSH authentication attempts.
- Understand DigitalOcean's recovery console and server recovery procedures.

## Table of Contents

1. Droplet Provisioning
2. SSH Key Generation and Authentication
3. Initial Login and System Updates
4. Non-Root User and SSH Hardening
5. SSH Client Configuration
6. UFW Firewall Configuration
7. Fail2Ban Installation and Configuration
8. DigitalOcean Recovery Console
9. Final Security Verification Checklist

---

# 1. Droplet Provisioning

## 1.1 What is a Droplet?

A Droplet is a virtual machine provided by DigitalOcean. It provides compute resources, including CPU, memory, storage, networking, and an operating system.

For this lab, we will deploy an Ubuntu server and progressively harden it before hosting an application.

## 1.2 Create the Droplet

In the DigitalOcean control panel:

1. Select **Create → Droplets**.
2. Choose a suitable Ubuntu LTS image.
3. Select a region close to your intended users or services.
4. Choose an appropriate CPU, memory, and storage configuration.
5. Configure SSH-key authentication.
6. Assign a meaningful hostname, such as `secure-web-01`.
7. Review the configuration and create the Droplet.

For a basic learning lab, a small instance is usually sufficient. Select the resources according to your workload and budget.

## 1.3 Security Considerations

- Prefer SSH public-key authentication over password-based SSH access.
- Use a strong SSH private-key passphrase for interactive administration.
- Keep the private key on your local machine.
- Restrict inbound network traffic to the ports required by your services.
- Take a snapshot or backup before major security changes.
- Avoid exposing application administration interfaces directly to the internet.

**Expected result:** An Ubuntu server is running and has an assigned public IP address.

Record the IP address for the following steps.

---

# 2. SSH Key Generation and Authentication

## 2.1 What Is SSH?

Secure Shell (SSH) is a protocol used to securely access and administer remote systems over an untrusted network.

SSH provides:

- Encrypted communication.
- Server identity verification.
- User authentication.
- Protection for administrative sessions against network interception.

SSH keys enable authentication without requiring the user to send an account password to the server.

## 2.2 Generate an SSH Key Pair

Run these commands on your **local computer**, not on the Droplet.

### Recommended: Ed25519

```bash
ssh-keygen -t ed25519 -a 100 -C "your_email@example.com"
```

When prompted:

1. Specify a file location or press Enter to use the default.
2. Enter a strong passphrase.
3. Confirm the passphrase.

The command creates two files:

| File                    | Purpose                                    |
| ----------------------- | ------------------------------------------ |
| `~/.ssh/id_ed25519`     | Private key; must remain secret            |
| `~/.ssh/id_ed25519.pub` | Public key; can be installed on the server |

Ed25519 is a strong, efficient choice for modern SSH deployments.

### Alternative: RSA

If compatibility with older systems is required, generate an RSA key:

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

This generates an RSA private key and its corresponding public key.

**Best practice:** Prefer Ed25519 for modern systems unless your environment requires a different supported algorithm.

## 2.3 Understand the Public-Key Authentication Model

SSH authentication uses a public/private key pair.

- **Private key:** Remains on the client machine and proves possession of the credential.
- **Public key:** Is installed in the server user's `~/.ssh/authorized_keys` file.
- **SSH server:** Verifies that the client possesses the private key corresponding to an authorized public key.

The public key does not need to be secret. The private key must never be copied to the server or shared with other people.

## 2.4 SSH Authentication Workflow

1. The client initiates an SSH connection to the server.
2. The server presents its host key so the client can verify the server's identity.
3. SSH negotiates the session's cryptographic algorithms and establishes encrypted communication.
4. The client requests authentication as a specific server user.
5. The server checks whether the user's account authorizes the corresponding public key.
6. The client proves possession of the private key by signing authentication data.
7. The server verifies the signature.
8. If verification and the applicable access checks succeed, the user is authenticated.

**Important distinction:** The server does not need the client's private key. It verifies the client's proof using the public key.

## 2.5 Register the Public Key with DigitalOcean

Display your public key:

```bash
cat ~/.ssh/id_ed25519.pub
```

Copy the complete output and add it to the SSH-key section when creating the Droplet.

If the Droplet already exists, you can install the public key through an authenticated administrative session.

Never upload or share `id_ed25519` itself. That is your private key.

## 2.6 Protect the Private Key

Check the local SSH directory:

```bash
ls -la ~/.ssh
```

For a typical private key, use restrictive permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

Use a passphrase for interactive keys. For automation, use appropriately scoped credentials and controlled secret management rather than leaving private keys exposed.

**Expected result:** Your local machine has a protected private key, and DigitalOcean has the corresponding public key.

---

# 3. Initial Login and System Updates

## 3.1 Connect to the Droplet

Replace the placeholder with your actual IP address:

```bash
ssh -i ~/.ssh/id_ed25519 root@YOUR_DROPLET_IP
```

On the first connection, SSH may ask you to verify the server's host-key fingerprint.

Verify the fingerprint using a trusted DigitalOcean console or other trusted provisioning channel before accepting it. Do not blindly trust an unexpected host-key change.

## 3.2 Update Package Information

Once connected, update the package index:

```bash
apt update
```

This refreshes the local package metadata from the configured repositories.

## 3.3 Install Available Updates

```bash
apt upgrade
```

Review the proposed changes and proceed when appropriate.

For an automated lab environment, `apt upgrade -y` can be used when you understand and accept the changes.

## 3.4 Check for a Required Reboot

After the updates, check whether Ubuntu indicates that a reboot is required:

```bash
if [ -f /var/run/reboot-required ]; then
    echo "A reboot is required."
else
    echo "No reboot is currently indicated."
fi
```

If required, reboot:

```bash
reboot
```

The SSH connection will disconnect. Wait for the server to restart, then reconnect.

## 3.5 Verify the System

After reconnecting, check the OS version and system uptime:

```bash
cat /etc/os-release
uptime
```

**Expected result:** The server is running an updated Ubuntu installation.

---

# 4. Create a Non-Root Administrative User and Harden SSH

## 4.1 Why Avoid Routine Root Access?

The root account has unrestricted administrative privileges. Routine work under root increases the potential impact of accidental commands and compromised sessions.

A non-root administrative account provides a safer default. Administrative actions can be explicitly elevated through `sudo`.

## 4.2 Create an Administrative User

While logged in as root:

```bash
adduser deployadmin
```

Replace `deployadmin` with your preferred username.

Follow the prompts to configure the account.

Add the user to the `sudo` group:

```bash
usermod -aG sudo deployadmin
```

The `-aG` flags append the user to the specified supplementary group without removing existing group memberships.

## 4.3 Install the Public Key for the New User

Do not copy the entire `/root/.ssh` directory. Instead, install only the intended public key.

Create the SSH directory:

```bash
install -d -m 700 -o deployadmin -g deployadmin \
  /home/deployadmin/.ssh
```

If the authorized public key is available on the server as `/root/id_ed25519.pub`, install it with:

```bash
install -m 600 -o deployadmin -g deployadmin \
  /root/id_ed25519.pub \
  /home/deployadmin/.ssh/authorized_keys
```

In this example, `/root/id_ed25519.pub` is a placeholder path: use the actual location of the public-key file.

Alternatively, transfer the public key through a trusted channel and append it to `authorized_keys`, taking care not to overwrite existing authorized keys.

Verify permissions:

```bash
ls -ld /home/deployadmin/.ssh
ls -l /home/deployadmin/.ssh/authorized_keys
```

The SSH directory should be owned by `deployadmin` and have mode `700`; `authorized_keys` should be owned by that user and have mode `600`.

## 4.4 Test the New User Before Changing SSH Settings

**Keep your existing root session open.** Open a second terminal on your local computer and test the new account:

```bash
ssh -i ~/.ssh/id_ed25519 deployadmin@YOUR_DROPLET_IP
```

Once connected, verify the user identity:

```bash
whoami
```

Expected output:

```text
deployadmin
```

Test administrative access:

```bash
sudo whoami
```

Expected output:

```text
root
```

If this fails, fix the account, key installation, or sudo permissions before proceeding.

## 4.5 Back Up the SSH Server Configuration

From the new account:

```bash
sudo cp -a /etc/ssh/sshd_config \
  /etc/ssh/sshd_config.backup
```

This preserves a copy of the existing configuration before modification.

## 4.6 Configure SSH Hardening

Open the SSH daemon configuration:

```bash
sudo nano /etc/ssh/sshd_config
```

Set the following directives:

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

These settings:

- Disable SSH login directly as root.
- Disable password-based SSH authentication.
- Enable public-key authentication.

Check for conflicting directives, included configuration files, and `Match` blocks. On Ubuntu, settings may also be defined in `/etc/ssh/sshd_config.d/`. Verify the effective configuration rather than assuming the last visible line always determines the result.

Save the file and exit the editor.

## 4.7 Validate the Configuration Before Applying It

Run:

```bash
sudo sshd -t
```

No output indicates that the configuration passed the syntax check.

If errors appear, correct them before proceeding.

Inspect the effective settings for the relevant connection context:

```bash
sudo sshd -T | grep -E \
  'permitrootlogin|passwordauthentication|pubkeyauthentication'
```

Expected values include:

```text
permitrootlogin no
passwordauthentication no
pubkeyauthentication yes
```

## 4.8 Apply the Configuration Safely

On Ubuntu, the SSH service is commonly named `ssh`.

```bash
sudo systemctl reload ssh
```

If reloading is unavailable or the service requires a restart, use:

```bash
sudo systemctl restart ssh
```

Keep the existing administrative session open. From a separate terminal, establish a new connection as `deployadmin` and verify that `sudo` still works.

Only after successful testing should you close the original session.

**Security checkpoint:** Root SSH access is disabled, and password-based SSH authentication is disabled. Console access and the provider's recovery mechanism remain important safeguards.

---

# 5. Simplify SSH Access with a Client Configuration

Typing a full SSH command repeatedly is unnecessary. The local SSH client supports named host aliases.

## 5.1 Create the SSH Client Configuration

Run this command on your **local computer**:

```bash
nano ~/.ssh/config
```

Add:

```text
Host my-website
    HostName YOUR_DROPLET_IP
    User deployadmin
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

Replace `YOUR_DROPLET_IP` with the server's IP address.

Save and exit.

## 5.2 Protect the Configuration File

```bash
chmod 600 ~/.ssh/config
```

## 5.3 Connect Using the Alias

Instead of entering the full SSH command:

```bash
ssh -i ~/.ssh/id_ed25519 \
  deployadmin@YOUR_DROPLET_IP
```

You can now run:

```bash
ssh my-website
```

The client reads the alias and applies the configured hostname, username, and private-key path.

**Expected result:** You can securely connect to your server using a short, memorable command.

---

# 6. Configure UFW (Uncomplicated Firewall)

## 6.1 What Is UFW?

UFW is a firewall management utility commonly used on Ubuntu. It provides a simpler interface for configuring Linux firewall rules.

A firewall reduces the server's exposed network surface by allowing only the traffic needed for its intended purpose.

## 6.2 Check the Existing Firewall Status

```bash
sudo ufw status verbose
```

If the firewall is inactive, configure the required access rules before enabling it.

## 6.3 Set Secure Default Policies

For a server intended to accept SSH and selected inbound application traffic:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

This denies unsolicited inbound traffic by default while allowing outbound connections.

## 6.4 Allow SSH Before Enabling the Firewall

```bash
sudo ufw allow OpenSSH
```

This uses the OpenSSH application profile if it is available on the system.

If SSH listens on a custom port, explicitly allow that port instead, using the correct protocol and port number.

For example, for SSH on TCP port 2222:

```bash
sudo ufw allow 2222/tcp
```

Do not add this example rule unless your SSH daemon actually listens on that port.

## 6.5 Enable the Firewall

Before enabling UFW, verify that the SSH rule matches your server's actual listening port and that you have a working alternative recovery method.

Then run:

```bash
sudo ufw enable
```

Confirm the rules:

```bash
sudo ufw status numbered
```

## 6.6 Allow Only Required Application Ports

For a web server, the required rules might include:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

Only add these rules if the server is intended to serve HTTP and HTTPS traffic.

Do not expose database ports, internal management services, or application-specific ports publicly unless there is a documented requirement. For sensitive services, prefer private networking or source-IP restrictions.

## 6.7 Verify Remote Access

From a separate local terminal, open a new SSH session:

```bash
ssh my-website
```

Confirm that the connection succeeds.

**Expected result:** UFW is active and only the necessary inbound traffic is permitted by the configured rules.

---

# 7. Install and Configure Fail2Ban

## 7.1 What Is Fail2Ban?

Fail2Ban monitors authentication logs and other configured log sources for repeated suspicious activity. When a configured rule is triggered, it can temporarily block the source IP using a firewall action.

Fail2Ban is a supplementary defensive measure, not a replacement for SSH key authentication, firewall restrictions, updates, or monitoring.

## 7.2 Install Fail2Ban

```bash
sudo apt update
sudo apt install fail2ban -y
```

Verify the service:

```bash
sudo systemctl status fail2ban
```

## 7.3 Configure an SSH Protection Jail

Create a local configuration file:

```bash
sudo nano /etc/fail2ban/jail.local
```

Add the following:

```ini
[sshd]
enabled = true
findtime = 10m
maxretry = 5
bantime = 1d
```

### Configuration Explanation

| Setting          | Purpose                                          |
| ---------------- | ------------------------------------------------ |
| `enabled = true` | Enables the SSH jail                             |
| `findtime = 10m` | Defines the period in which failures are counted |
| `maxretry = 5`   | Sets the configured retry threshold              |
| `bantime = 1d`   | Sets a ban duration of one day                   |

These settings configure the jail's intended thresholds. The exact behavior depends on the installed Fail2Ban version, filters, logging backend, and effective configuration.

**Best practice:** Keep custom settings in local override files rather than modifying packaged defaults directly.

## 7.4 Restart and Enable the Service

```bash
sudo systemctl enable --now fail2ban
sudo systemctl restart fail2ban
```

Check the service:

```bash
sudo systemctl status fail2ban
```

## 7.5 Verify the SSH Jail

Check the jail status:

```bash
sudo fail2ban-client status
```

Inspect the SSH jail:

```bash
sudo fail2ban-client status sshd
```

Verify the configured ban time:

```bash
sudo fail2ban-client get sshd bantime
```

For a one-day ban, the result should be:

```text
86400
```

seconds.

## 7.6 Understand the Defense Workflow

1. A remote client attempts to authenticate.
2. Failed authentication attempts are recorded in the system logs.
3. Fail2Ban reads the relevant logs using the configured backend.
4. The SSH jail evaluates the failures against its filter and thresholds.
5. When the configured threshold is reached, Fail2Ban invokes its firewall action.
6. The source IP is blocked for the configured duration, subject to the firewall action and environment.

On Ubuntu, authentication messages may be available through the system journal rather than a traditional `/var/log/auth.log` file, depending on the logging configuration.

## 7.7 Operational Considerations

- Do not repeatedly trigger failed logins against your own server to test bans.
- Confirm that Fail2Ban uses an appropriate firewall action for the host.
- Consider trusted administrative IPs carefully; overly broad exclusions reduce protection.
- Remember that banning IP addresses can affect users behind shared NAT or dynamic IP addresses.
- Keep the system and Fail2Ban package updated.

**Expected result:** The SSH jail is enabled, its status can be inspected, and repeated authentication failures can trigger configured bans.

---

# 8. DigitalOcean Recovery Console

## 8.1 Why Recovery Access Matters

Server hardening changes can accidentally interrupt remote access. Examples include incorrect SSH configuration, firewall rules that block the SSH port, or incorrect file permissions.

The DigitalOcean Recovery Console provides an alternative access path for troubleshooting supported Droplet issues when ordinary SSH access is unavailable.

## 8.2 Recovery Procedure

If you lose SSH access:

1. Open the DigitalOcean control panel.
2. Select the affected Droplet.
3. Locate its console or recovery options.
4. Open the available console and authenticate as required.
5. Inspect the SSH service, firewall rules, user permissions, and configuration files.
6. Correct the underlying problem.
7. Validate the SSH configuration and restore remote connectivity.

Useful diagnostic commands include:

```bash
systemctl status ssh
ss -tlnp
ufw status verbose
sshd -t
journalctl -u ssh --no-pager -n 100
```

For authentication-related issues, also inspect the relevant system authentication logs.

The exact recovery workflow depends on the failure type and the options available for the Droplet. A console is not a substitute for maintaining tested backups.

## 8.3 Recovery Best Practices

- Maintain a tested backup or snapshot before significant changes.
- Keep an administrative SSH session open while modifying access controls.
- Test a second SSH session after each major security change.
- Verify configuration syntax before reloading or restarting SSH.
- Know how to access the provider's recovery facilities before you need them.

---

# 9. Final Security Verification Checklist

Use this checklist to verify the completed lab.

- [ ] Ubuntu Droplet is provisioned and updated.
- [ ] SSH key pair is generated on the local computer.
- [ ] Private key is protected and never copied to the server.
- [ ] Server host-key identity is verified.
- [ ] Non-root administrative user exists.
- [ ] Public-key login works for the new user.
- [ ] `sudo` administrative access has been tested.
- [ ] `PermitRootLogin no` is effective.
- [ ] `PasswordAuthentication no` is effective.
- [ ] SSH configuration passes `sshd -t`.
- [ ] SSH client alias works from the local computer.
- [ ] UFW is active with the required inbound rules.
- [ ] Fail2Ban is running and the SSH jail is enabled.
- [ ] Recovery Console access and backup procedures are understood.

## Final Outcome

At the end of Module 1, you should have a hardened Ubuntu server with key-based SSH access, a non-root administrative account, restricted inbound network access, basic automated defenses against repeated SSH authentication failures, and a recovery plan.

**Core principle:** Secure access first, verify every change, and only then enforce restrictions. This reduces unnecessary exposure while minimizing the risk of accidental administrative lockout.
