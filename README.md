# PaperMC Infrastructure Deployment

This repository contains the Infrastructure as Code (IaC) configuration for deploying and managing a PaperMC Minecraft server. The deployment is fully automated using Ansible and GitHub Actions, with automated deduplicated backups routed to a Hetzner Storage Box.

## 🏗️ Architecture Overview

* **Server Software:** PaperMC (Java) managed via systemd.
* **Hosting Environment:** Hetzner Cloud (VPS) / Local Home Server.
* **Configuration Management:** Ansible.
* **Continuous Deployment (CD):** GitHub Actions.
* **Backup Solution:** BorgBackup (block-level deduplication) utilizing a Grandfather-Father-Son (GFS) retention policy over SSH (Port 23).
* **Storage Target:** Hetzner Storage Box.

## 📂 Repository Structure

```text
papermc-ansible/
├── .github/workflows/deploy.yml   # GitHub Actions pipeline definition
├── inventory/
│   ├── vps_production.yml         # Cloud environment targets
│   └── home_production.yml        # Local environment targets
├── group_vars/
│   ├── all.yml                    # Global variables (timezone, ports)
│   └── minecraft.yml              # Service variables (RAM allocation)
├── roles/
│   ├── system/                    # Base OS, firewall, and dependencies
│   ├── minecraft/                 # PaperMC binary, EULA, and systemd service
│   └── backup/                    # BorgBackup scripts and cron scheduling
├── playbook.yml                   # Main execution playbook
├── ansible.cfg                    # Local Ansible configuration
└── README.md                      # Project documentation