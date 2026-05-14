# PaperMC Infrastructure Deployment

Ansible playbook for PaperMC Minecraft server provisioning. Installs Java, configures UFW, deploys systemd service with Aikar's flags, and sets up Borg-based backup/restore via cron.

## Prerequisites

- Ansible installed on control machine
- Target server running Debian/Ubuntu with root SSH access
- Borg repository (SSH) for backups

## Setup

```bash
# 1. Configure your server address
nano inventory.yml

# 2. Set Borg passphrase, backup repo, and server paths
nano vars.yml

# 3. If you want to restore from the latest backup on first deploy,
#    keep restore_latest_backup: true in vars.yml.
#    Otherwise set it to false and start fresh with a new paper.jar.
```

## Deploy

```bash
ansible-playbook site.yml
```

## Usage

The playbook installs a systemd service and a daily cron backup at 02:00.

| Action | Command |
|---|---|
| Service status | `systemctl status minecraft` |
| View logs | `journalctl -u minecraft -f` |
| List backups | `sudo /opt/minecraft-scripts/restore-borg.sh --list` |
| Restore backup | `sudo /opt/minecraft-scripts/restore-borg.sh 2026-05-14` |

Backups use Borg (lz4 compression, excludes `cache/`). Retention: 14 daily, 8 weekly, 3 monthly.
