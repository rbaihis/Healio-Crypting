
# Automate Decrypt On boot 


##### Navigate For details:
- [Home](README.md)
- [Crypt Disk](/1.encrypt_luks.md)
- [Add Key - BackUp Header](/2.add_key_backup_header.md)
- [Booting Options - Automated vs Manual approach](/3_automate_decrypt_on_boot.md)
- [Risky-Crypt Without Formatting](/4.no_formatting_encryption.md)
 ---


## Option-1 - Passphrase required in boot
> [!IMPORTANT]
> - This configuration `requires system user to enter passphrase` when system is booting.
> - If using cloud, and you have no access to the VM is console without relying on SSH-access, `then do not use this`(unconvenient).

Edit `/etc/crypttab`:

```bash
echo "secure_volume /dev/sdX none luks" | sudo tee -a /etc/crypttab
```

Update `/etc/fstab`:

```bash
echo "/dev/mapper/secure_volume /mnt/secure ext4 defaults 0 2" | sudo tee -a /etc/fstab
```

---
## Option-2 - Auto-Boot No Manual Intervention
> [!IMPORTANT]
> - This configuration `requires storing the key in the file-system locally`  **less secure then option-1**.


Edit `/etc/crypttab`:

```bash
# requires a key already created and recognized by the current LUKS-Header
echo "secure_volume /dev/sdX /pathToKey luks" | sudo tee -a /etc/crypttab
```

Update `/etc/fstab`:

```bash
# making sure the mount point exists
mkdir -p /mnt/secure

echo "/dev/mapper/secure_volume /mnt/secure ext4 defaults 0 2" | sudo tee -a /etc/fstab
```


---
## Option-3 - cryptsetup-ssh for Remote LUKS Unlocking

### When to Use cryptsetup-ssh?
✅ You need remote administration of LUKS volumes.<br>
✅ You want to automate decryption in scripts/workflows.<br>
✅ Physical access to the machine is impossible or inconvenient.<br>


## Critical Notes
> [!WARNING]
> **Backup Passphrase:** Always keep a LUKS passphrase as a fallback:
```bash
sudo cryptsetup luksAddKey /dev/sdX
```

> [!WARNING]
> **SSH Key Security:** Store the private key (`luks_remote_key`) in a secure location (e.g., hardware token).

> [!WARNING]
> **Network Dependency:** Requires SSH access to the machine during boot (not suitable for offline systems).

> [!NOTE]
> **Key Feature:** Decrypt disks remotely using SSH instead of typing a passphrase on a local console.

> [!NOTE]
> **How It Works:** Integrates SSH authentication with LUKS, allowing you to use SSH keys or passwords to unlock encrypted drives.

### Use Cases
- **Remote Servers:** Unlock LUKS-encrypted root/filesystems on headless servers.
- **Disaster Recovery:** Access encrypted backups/storage from afar.
- **Automation:** Script-driven decryption in CI/CD pipelines or remote deployments.
- **IoT/Embedded Devices:** Manage encryption on devices without keyboards/displays.

### Benefits
| Advantage | Description |
|-----------|-------------|
| **No Physical Access Needed** | Unlock disks remotely via SSH. |
| **SSH Key Integration** | Leverage existing SSH key infrastructure for authentication. |
| **Audit Trails** | Track decryption attempts via SSH logs. |
| **Flexibility** | Works with LUKS1/LUKS2 and most Linux distributions. |

### Risks & Considerations
> [!CAUTION]
> **SSH Security:** If your SSH server is compromised, attackers could unlock the disk.

> [!CAUTION]
> **Single Point of Failure:** Losing SSH access means losing access to data (always keep a backup passphrase).

> [!CAUTION]
> **Complexity:** Requires proper SSH server/client configuration.

---

### Step 1: Install cryptsetup-ssh
```bash
# Debian/Ubuntu
sudo apt install cryptsetup-ssh

```

### Step 2: Configure SSH Server
Ensure `sshd` is running and allows key-based authentication:
```bash
# Edit SSH server config
sudo nano /etc/ssh/sshd_config

# Ensure these lines exist:
PubkeyAuthentication yes
PasswordAuthentication no  # Disable for security(if possible)
```
Restart SSH:
```bash
sudo systemctl restart sshd
```

### Step 3: Add SSH Key to LUKS Volume
Generate an SSH Key Pair (if you don’t have one):
```bash
ssh-keygen -t ed25519 -f ~/.ssh/luks_remote_key
```
Add the Public Key to LUKS:
```bash
#Replace `/dev/sdX` with your LUKS device (e.g., `/dev/nvme0n1p3`).

sudo cryptsetup-ssh add-keys /dev/sdX --ssh-key ~/.ssh/luks_remote_key.pub
```

### Step 4: Update /etc/crypttab
Configure the encrypted volume to use SSH unlocking:
```bash
# Edit /etc/crypttab
sudo nano /etc/crypttab

# Add this line (replace "encrypted_volume" with your mapper name)
encrypted_volume /dev/sdX none ssh,keyscript=/usr/share/cryptsetup-ssh/ssh-keyscript
```

### Step 5: Test Remote Unlocking
From a Remote Machine:
```bash
ssh -i ~/.ssh/luks_remote_key user@remote_host
```
The SSH connection will trigger the LUKS unlock process automatically.

Verify the Volume is Mapped:
```bash
lsblk /dev/sdX  # Check if "encrypted_volume" appears under mappers
```


### Official Docs
- **GitHub:** cryptsetup-ssh
- **man cryptsetup-ssh**


---
- [Home](README.md)
- [Crypt Disk](/1.encrypt_luks.md)
- [Add Key - BackUp Header](/2.add_key_backup_header.md)
- [Booting Options - Automated vs Manual approach](/3_automate_decrypt_on_boot.md)
- [Risky-Crypt Without Formatting](/4.no_formatting_encryption.md)

