# Minecraft Server Deployment with Borg Backup on Hetzner

This Ansible playbook automates the deployment of a Minecraft server on a Hetzner VPS, restoring server files from a Borg backup stored on a Hetzner Storage Box.

## Features

- 🚀 **Automated deployment** on fresh Hetzner VPS
- 🔄 **Restores from Borg backup** - no manual file downloads
- 🔐 **Ansible Vault** for secret management (Borg passphrase, repo URL)
- 🔥 **UFW firewall** configuration (SSH + Minecraft port)
- ⚙️ **Systemd service** for automatic server startup
- 💾 **Automated nightly backups** with retention policy
- 📝 **Comprehensive logging** for troubleshooting
- 🆕 **Borg repository initialization** playbook included

## Prerequisites

### Local Machine
- Ansible installed (>= 2.9)
- SSH access to your Hetzner VPS
- Hetzner Storage Box with Borg repository containing Minecraft server files
- Borg backup installed locally (for repo initialization)

### Hetzner Storage Box
- Active Storage Box with SSH access (port 23)
- SSH key already configured for passwordless access (or use password)

### Minecraft Server
- World size: ~10 GB (adjust storage estimates in playbook if different)

## Repository Structure

```
.
├── ansible.cfg
├── inventory.yml
├── LICENSE
├── README.md
├── playbooks/
│   ├── deploy-minecraft.yml
│   └── init-borg-repo.yml
└── templates/
  ├── backup.sh.j2
  └── minecraft.service.j2
```

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/Markus-Pasanen/papermc-ansible
cd papermc-ansible
```

### 2. Create Your Inventory File

Create an `inventory` file:

```ini
[minecraft]
your-vps-ip ansible_user=root
```

### 3. Prepare Your Secrets with Ansible Vault

Create an encrypted secrets file:

```bash
ansible-vault create secrets.yml
```

Add your Borg repository details:

```yaml
borg_repo: ssh://u123456@u123456.your-storagebox.de:23/./backups/minecraft
borg_passphrase: your-secret-borg-passphrase-here
```

**Important:** Replace with your actual Storage Box username and repository path.

## Borg Repository Initialization

Before deploying your Minecraft server, you need a Borg repository on your Storage Box. Use the initialization playbook:

### Initialize a New Repository

```bash
ansible-playbook init-borg-repo.yml --ask-vault-pass
```

This playbook will:
- Check if the repository already exists
- Initialize a new Borg repository with `repokey-blake2` encryption
- Set a **250 GB storage quota** (based on ~10 GB world size)
- Display repository information upon completion

### Storage Quota Calculation

Example based on 10 GB world and backup strategy (7 daily, 4 weekly, 6 monthly):

| Backup Type | Count | Est. Additional Data |
|------------|-------|---------------------|
| Daily | 7 | ~10-15 GB (daily changes) |
| Weekly | 4 | ~20-30 GB (weekly changes) |
| Monthly | 6 | ~40-60 GB (monthly changes) |
| **Raw Total** | **17 backups** | **~70-105 GB** |
| **With Borg deduplication** | **Same** | **~40-60 GB** |

**Quota set to 250 GB** - Provides safety buffer for world growth and backup retention adjustments.

To adjust the quota, modify the `--storage-quota` value in `init-borg-repo.yml`:
```yaml
--storage-quota 250G  # 250 gigabytes
--storage-quota 500G  # 500 gigabytes
--storage-quota 1T    # 1 terabyte
--storage-quota 0     # no quota
```

## Deploy the Minecraft Server

Once your Borg repository is initialized (or if you already have one with backups), run the main deployment:

```bash
ansible-playbook -i inventory deploy.yml --ask-vault-pass
```

## What Gets Deployed

| Component | Location | Description |
|-----------|----------|-------------|
| Minecraft Server | `/opt/minecraft` | Server files restored from backup |
| Minecraft User | `minecraft` | System user running the server |
| Systemd Service | `/etc/systemd/system/minecraft.service` | Auto-starts server on boot |
| Borg SSH Key | `/root/.ssh/borg_backup` | SSH key for Storage Box access |
| Backup Script | `/usr/local/bin/minecraft_backup.sh` | Nightly backup script |
| Backup Log | `/var/log/minecraft_backup.log` | Backup operation logs |
| Cron Job | `/etc/cron.d/ansible_minecraft_backup` | Runs backup daily at 2 AM |

## Firewall Configuration

The playbook configures UFW to:
- Allow SSH (port 22)
- Allow Minecraft server (port 25565 by default)
- Deny all other incoming traffic

To change the Minecraft port, modify the `server_port` variable in `deploy.yml`.

## Backup Retention Policy

Nightly backups follow this retention schedule:
- **7 daily** backups
- **4 weekly** backups
- **6 monthly** backups

Backups exclude:
- Log files (`*.log`, `logs/`)
- Cache directories (`cache/`)
- Temporary files (`tmp/`)

## Manual Operations

### Check Backup Status

```bash
# View backup logs
sudo tail -f /var/log/minecraft_backup.log

# List available backups
sudo borg list {{ borg_repo }}
```

### Run Backup Manually

```bash
sudo /usr/local/bin/minecraft_backup.sh
```

### Restore from Specific Backup

```bash
# List backups to find archive name
sudo borg list {{ borg_repo }}

# Restore specific archive
sudo borg extract {{ borg_repo }}::hostname-2024-01-15_02:00:00 --destination /tmp/restore
```

### Monitor Storage Usage

```bash
# Check repository size and usage
sudo borg info {{ borg_repo }}

# List all backups with sizes
sudo borg list --format '{archive:<36} {time} {size:<10} {num_files} files\n' {{ borg_repo }}
```

### Manage Minecraft Server

```bash
# Check server status
sudo systemctl status minecraft

# Start/stop/restart server
sudo systemctl stop minecraft
sudo systemctl start minecraft
sudo systemctl restart minecraft

# View server logs
sudo journalctl -u minecraft -f
```

## Customization

### Change Minecraft Version

Your Borg backup determines the Minecraft version. To upgrade:
1. Stop the server
2. Update files in `/opt/minecraft`
3. Restart the server
4. The next nightly backup will save the new version

### Modify Backup Schedule

Edit the cron task in `deploy.yml`:

```yaml
- name: Schedule nightly backup via cron
  cron:
    name: "Minecraft server nightly backup"
    minute: "0"
    hour: "2"           # Change to desired hour (UTC)
    job: "/usr/local/bin/minecraft_backup.sh >> /var/log/minecraft_backup.log 2>&1"
```

### Adjust Backup Retention

Modify the `borg prune` command in `backup.sh.j2`:

```bash
borg prune                              \
    --keep-daily 7      \  # Change number of daily backups
    --keep-weekly 4     \  # Change number of weekly backups
    --keep-monthly 6       # Change number of monthly backups
```

## Troubleshooting

### Borg Repository Not Accessible

```bash
# Test repository access
BORG_PASSPHRASE='your-passphrase' borg list {{ borg_repo }}

# Check SSH connection to Storage Box
ssh -p23 u123456@u123456.your-storagebox.de
```

### Repository Quota Issues

If you hit the storage quota:

```bash
# Check current usage
borg info {{ borg_repo }}

# Options:
# 1. Prune old backups manually
# 2. Increase quota in init-borg-repo.yml and re-run
# 3. Contact Hetzner to upgrade Storage Box
```

### Minecraft Server Won't Start

```bash
# Check service status
sudo systemctl status minecraft

# View detailed logs
sudo journalctl -u minecraft -n 50 --no-pager

# Check file permissions
ls -la /opt/minecraft/
sudo -u minecraft java -version  # Verify Java access
```

## Security Considerations

- **Secrets**: All sensitive data encrypted with Ansible Vault
- **Firewall**: Default deny policy with only necessary ports open
- **Systemd Hardening**: Service runs with `PrivateTmp` and `NoNewPrivileges`
- **Backup Encryption**: Borg repository encrypted with repokey-blake2
- **Regular Updates**: Keep system packages updated periodically

## Contributing

Feel free to submit issues and enhancement requests!

## License

MIT License - feel free to use this for your own Minecraft servers.

## Acknowledgments

- [Hetzner](https://www.hetzner.com) for reliable VPS and Storage Box services
- [BorgBackup](https://www.borgbackup.org) for efficient deduplicating backups
- Minecraft community for keeping the game alive