# GOADURR: Game of Active Directory containerized 🐐🏰🐳
> **AD Lab in a box.**

## 🤷 But why?

Do you want to play around with the beauty that is **Micro$oft Active Directory** but your mind goes blank, when thinking about setting up lab ranges?  
Do words like `Ansible`, `Terraform`, and `Proxmox` **make you enter flight, not fight mode**?  
Do you dislike running a mini server 24/7 just for the **5 minutes per week that you'll actually use it**?

**Then this is for you.**  
GOADURR creates a full-featured AD lab for pentest practice using `docker compose`. It spawns QEMU/KVM containers that run the actual Windows VMs, handling all the messy networking glue and machine provisioning so you don't have to.

## 🐧 Target audience
* **Linux/WSL Users:** Set up GOAD on your main device, which is Linux anyway, right? WSL might work, but is currently untested.
* **Casual Labbers:** Spin up the lab when you need it, kill it when you don't.
* **Container Enthusiasts:** People who believe everything looks better inside a container.

---

## 🚀 Quick Start

### 1. Requirements
**Warning:** This project adds about **25GB** of downloads to the existing GOAD requirements (each container pulls its own Windows ISO).

**Hardware Specs:**
| Component | Minimum Req | Notes |
| :--- | :--- | :--- |
| **CPU** | ~8 Cores | 5 VMs x 2 vCores recommended. <br> Works with less but slower |
| **RAM** | 24GB+ | 5 VMs x 4GB + Overhead |
| **Disk** | 200GB+ | 5 VMs x 40GB Storage |
| **OS** | Linux / WSL2 | Must support KVM virtualization |
| **Time** | 2-3 hours | Initial setup time. <br> Depends on download, CPU and disk speed |

> [!Warning] Check power settings  
> Don’t let your host go to sleep during the first run.

### 2. Run it
```bash
# Clone the repo
git clone https://github.com/ripm4ge/goadurr
cd goadurr

# Start GOADURR
docker compose up -d
docker compose logs -f --tail=200

# Start attack box
docker compose --profile attackbox up -d kali
```

**Provisioning is done** when the `provisioner` container shows the following output:

```bash
docker compose logs provisioner

# shows the following after successful initial setup
goad-provisioner  | [+] Provisioning completed successfully!
goad-provisioner  | [+] Marker file created. Future runs will be skipped.
```
(Alternative check: `docker compose ps` → `provisioner` should eventually be `exited (0)`.)

## 💻 How do I connect?
This lab exposes local web UIs (open from your host browser):
- DC01 console (troubleshooting): `http://localhost:8006`
- Kali desktop (browser VNC): `https://localhost:6901` 
- RDP from host to VM IPs (check table below for VM IP list)

Kali credentials: `kasm_user:password`  
Windows setup credentials: `Docker:admin`

> [!Warning] Exposure  
> Do **not** expose these ports to the internet. Change the VNC password before use.  
> To be extra safe, bind ports to localhost only, e.g.`127.0.0.1:8006:8006`. 


## 🐐 What is this GOAD you speak of?
Game of Active Directory [Orange-Cyberdefense/GOAD](https://github.com/Orange-Cyberdefense/GOAD) is an AD lab environment for pentesting practice. It features multiple domains/servers and intentionally vulnerable configurations to learn common AD attacks.

It consists of the following systems + a provisioner and attacker box added for GOADURR:

| Role | Service | Hostname | Container IP | VM IP | 
|------|---------|----------|----|----|
| Domain Controller | `dc01` | `kingslanding` | `192.168.56.110` | `192.168.56.10` |
| Domain Controller | `dc02` | `winterfell` | `192.168.56.111` | `192.168.56.11` |
| Domain Controller | `dc03` | `meereen` | `192.168.56.112` | `192.168.56.12` |
| Member server | `srv01` | `castelblack` | `192.168.56.122` | `192.168.56.22` |
| Member server | `srv02` | `braavos` | `192.168.56.123` | `192.168.56.23` |
| Provisioner | `provisioner` | - | `192.168.56.100` | - |
| Attacker box | `kali` | - | `192.168.56.50` | - |

Docker network: `goad_bridge` = `192.168.56.0/24`

> [!Info] What now?   
> If you feel lost, check out the writeup from the GOAD developer [mayfly277](https://mayfly277.github.io/categories/goad/), or any other GOAD walkthrough.

## ⚙️ Technical overview

The heavy lifting is done by [dockur/windows](https://github.com/dockur/windows). The project uses QEMU/KVM inside Docker containers to spin up Windows VMs. It is itself based on the [qemus/qemu](https://github.com/qemus/qemu) container project.

**The networking problem:** default qemu/dockur containers use NAT, and Active Directory hates NAT.
- *Why?* Domain Controllers (DCs) register the internal QEMU VM IP (e.g., 172.x) in DNS. When other machines try to talk to the DC, they get the NAT'd, unreachable IP, causing all kinds of headaches.
- *Solution:* Modifying the container entrypoint to set up a **bridge + tap** interface so the VM can bridge directly onto the Docker network. This avoids host-side `macvlan` complexity and ensures domain controllers use the correct IP and DNS settings.


## 🛠️ Maintenance & Troubleshooting
As with the original GOAD, if a task fails (`Ansible`), simply restarting the provisioner usually fixes it:
```bash
docker compose up provisioner
```

**Useful Commands**  

Rerun Provisioner:
```bash
docker compose run --rm --entrypoint "rm -f /goad/data/.goad_provisioned" provisioner
docker compose up provisioner
```
Run manual Ansible command:
```bash
docker compose run --rm --entrypoint \
  "ansible dc02 -i /inventory/inventory.ini -m ansible.windows.win_shell -a \"Install-WindowsFeature RSAT-DNS-Server; Import-Module DnsServer\"" \
  provisioner
```

---
## 🤝 Acknowledgments
This project is basic glue between these awesome projects — go check them out:

- **GOAD** [Orange-Cyberdefense/GOAD](https://github.com/Orange-Cyberdefense/GOAD)
- **Dockur Windows** [dockur/windows](https://github.com/dockur/windows)


---

## TODO:
- [ ] test WSL
- [ ] test podman
- [ ] fix 4x ISO download (e.g. by adding downloader side container and bind mounting ISO path)

