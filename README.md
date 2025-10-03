# Slurm HPC Cluster Automation with Ansible

## Overview
This project represents my journey into learning Ansible by tackling a real-world challenge: automating the complex deployment of a Slurm HPC cluster. I took my existing manual installation script and transformed it into a dynamic, enterprise-grade Ansible automation. Now, you can deploy a production-ready Slurm cluster where the version is dynamically fetched from the official website. For a deeper dive into the manual process that inspired this automation, check out my Medium article: A Comprehensive Guide to Installing Slurm from Source https://medium.com/@vishal.papan/a-comprehensive-guide-to-installing-slurm-from-source-20028ff837d7.

## Architecture Diagram

```
                    ┌─────────────────┐
                    │   Ansible       │
                    │   Control Node  │
                    │                 │
                    │ • Playbooks     │
                    │ • Roles         │
                    │ • Templates     │
                    └─────────────────┘
                            │
                            ▼
    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
    │  Controller     │    │  Compute Node   │    │  Compute Node   │
    │  Node           │    │                 │    │                 │
    │                 │    │                 │    │                 │
    │ • slurmctld     │    │ • slurmd        │    │ • slurmd        │
    │ • slurmdbd      │    │ • Auto-config   │    │ • Auto-config   │
    │ • MariaDB       │    │ • Munge auth    │    │ • Munge auth    │
    │ • Munge auth    │    └─────────────────┘    └─────────────────┘
    └─────────────────┘
```

## Key Features

-  **Dynamic Configuration**: Auto-discovery of all cluster nodes
-  **cgroup v2 Compatible**: Modern OS support (Amazon Linux 2023+)
-  **Version Agnostic**: Easy Slurm version changes via variables
-  **Modular Design**: 10+ reusable Ansible roles
-  **Multi-Node Support**: Controller + Compute node architecture
-  **Idempotent**: Safe for multiple runs
-  **Production Ready**: Error handling and service management

## 📁 Project Structure

```
slurm-ansible/
├── deploy-slurm-controller.yml    # Controller deployment playbook
├── deploy-slurm-compute.yml       # Compute node deployment playbook
├── inventory.ini                  # Host definitions
├── munge.key                      # Shared authentication key
└── roles/
    ├── slurm_base/               # Directory structure
    ├── slurm_user/               # User & permissions
    ├── munge/                    # Authentication service
    ├── build_dependencies/       # Compilation tools
    ├── mariadb/                  # Database (controllers only)
    ├── json_c/                   # JSON-C from source
    ├── libjwt/                   # JWT library from source
    ├── slurm_install/            # Slurm compilation & install
    └── slurm_config/             # Dynamic configuration templates
```

##  Quick Start

### 1. Prerequisites
```bash
# Install Ansible
pip install ansible

# Clone repository
git clone https://github.com/vishalpapan/ansible-slurm-deployment.git
cd slurm-ansible
```

### 2. Configure Inventory
Edit `inventory.ini`:
```ini
[slurm_controllers]
control1 ansible_host=10.0.2.68 ansible_user=ec2-user

[slurm_computes]
compute1 ansible_host=10.0.14.85 ansible_user=ec2-user
compute2 ansible_host=10.0.15.11 ansible_user=ec2-user
```

### 3. Deploy Controller
```bash
ansible-playbook -i inventory.ini deploy-slurm-controller.yml
```

### 4. Deploy Compute Nodes
```bash
ansible-playbook -i inventory.ini deploy-slurm-compute.yml
```

##  Configuration Variables

### Global Variables (roles/slurm_install/defaults/main.yml)
```yaml
slurm_version: "25.05.3"                    # Change Slurm version here
slurm_download_url: "https://download.schedmd.com/slurm/slurm-{{ slurm_version }}.tar.bz2"
```

### Cluster Configuration (roles/slurm_config/defaults/main.yml)
```yaml
slurm_primary_controller: "control1"        # Main controller hostname
slurm_cluster_name: "Slurm-Cluster"        # Cluster name
```

##  Version Management

### Upgrading Slurm Version
1. Edit `roles/slurm_install/defaults/main.yml`
2. Change `slurm_version: "25.05.3"` to desired version
3. Re-run playbooks

### Supported Versions
- Slurm 23.x.x and above
- Tested on: 25.05.3, 24.05.x, 23.11.x

##  Dynamic Features

### Auto Node Discovery
The system automatically discovers all compute nodes from inventory:
```jinja2
# In slurm.conf.j2 template
{% for host in groups['slurm_computes'] %}
NodeName={{ hostvars[host]['ansible_hostname'] }} CPUs={{ hostvars[host].ansible_processor_vcpus }} State=UNKNOWN
{% endfor %}
```

### Automatic Partition Creation
```jinja2
PartitionName=debug Nodes={% for host in groups['slurm_computes'] %}{{ hostvars[host]['ansible_hostname'] }}{% if not loop.last %},{% endif %}{% endfor %} Default=YES MaxTime=INFINITE State=UP
```


## Advanced Configuration

### Custom Build Options
Edit `roles/slurm_install/tasks/main.yml` configure command:
```bash
./configure --prefix=/usr \
  --sysconfdir=/etc/slurm \
  --with-mysql_config=/usr/bin \
  --enable-cgroupv2 \          # cgroup v2 support
  --with-jwt=/usr/local \      # JWT authentication
  --with-json=/usr/local       # JSON support
```

### Adding New Compute Nodes
1. Add to `inventory.ini` under `[slurm_computes]`
2. Run: `ansible-playbook -i inventory.ini deploy-slurm-compute.yml --limit new_node`
3. Restart slurmctld: `systemctl restart slurmctld`

##Monitoring & Troubleshooting

### Service Status
```bash
# Check all services
systemctl status slurmctld slurmdbd slurmd munge

# View logs
journalctl -u slurmctld -f
tail -f /var/log/slurm/slurmctld.log
```

### Cluster Health
```bash
# Check cluster status
sinfo
squeue
scontrol show nodes
```

## 🚨 Known Issues & Solutions

### cgroup v2 Compatibility
**Issue**: Modern OS uses cgroup v2, Slurm defaults to v1
**Solution**: `--enable-cgroupv2` flag + proper cgroup.conf

### Dependency Resolution
**Issue**: Missing build dependencies
**Solution**: Comprehensive package installation in `build_dependencies` role

## Contributing

1. Fork the repository
2. Create feature branch
3. Test on clean environment
4. Submit pull request

## 📄 License

MIT License - See LICENSE file for details

## You can connect with me 

- LinkedIn: https://www.linkedin.com/in/vishal-papan

---
