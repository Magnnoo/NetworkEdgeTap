# NetworkEdgeTap

**An all-in-one network monitoring VM at the edge of a network to passively observe traffic between WAN and LAN, visualize flows, and prepare for IDS integration (Suricata/Zeek).**

---

## Inline Network Monitoring Lab — Setup Documentation

### Project Goal

Set up a single VM at the network edge to:

- Passively observe traffic between WAN and LAN  
- Visualize network flows  
- Prepare for IDS integration with tools like Suricata and Zeek  

---

## Hardware Used

- Lenovo ThinkCentre M910x Mini PC  
- Intel I350 4-port Gigabit NIC (used for WAN/LAN and monitoring)  
- Home/Enterprise router and modem (traffic source)  
- Proxmox VE installed on local storage  

---

## Software Used

- Proxmox VE (virtualization host)  
- Ubuntu 24.04.3 LTS VM (base for monitoring)  
- Docker + Docker Compose for containerized tools  
- ntopng container for network traffic visualization  

---

## Steps Completed

### 1. Install Proxmox

1. Download the Proxmox VE ISO from [Proxmox Download](https://www.proxmox.com/en/downloads)  
2. Boot the Mini PC from USB and install Proxmox to internal storage.  
3. Configure networking for the management interface (`vmbr0`) during installation.  

---

### 2. Create Ubuntu VM

1. In Proxmox Web GUI, create a new VM.  
2. Assign minimal resources:
   - **CPU:** 1 core  
   - **Memory:** 2 GB  
   - **Disk:** 12 GB  
3. Set OSType: Linux 64-bit.  
4. Boot from ISO or prebuilt Ubuntu template.  
5. **Do not boot the VM yet** — network configuration comes next.  

---

### 3. Configure Host Bridges in Proxmox

- **vmbr0:** Management bridge (Proxmox host + VM SSH access)  
- **vmbr1 (planned):** Inline bridge connecting WAN/LAN NICs  

Example configuration in `/etc/network/interfaces`:

```text
auto vmbr1
iface vmbr1 inet manual
    bridge-ports enp1s0f0 enp1s0f1
    bridge-stp off
    bridge-fd 0
```

- **enp1s0f0 → WAN**  
- **enp1s0f1 → LAN**  
- This bridge is passive and will mirror traffic into the VM.  
- **Note:** Physical cables are not connected at this stage.  

---

### 4. Assign Network Interfaces to VM

- **VM net0:** Management → `vmbr0`  
- **VM net1:** Monitor → `vmbr1`  

Example in `100.conf`:

```text
net0: virtio=<MAC>,bridge=vmbr0
net1: virtio=<MAC>,bridge=vmbr1
```

After boot, configure monitor interface in the VM:

```bash
sudo ip link set monitor0 up
sudo ip link set monitor0 promisc on
```

---

### 5. Install Docker & Docker Compose

```bash
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker
sudo systemctl start docker
```

---

### 6. Deploy ntopng Docker Container

1. Create project folder:

```bash
mkdir ~/ntop && cd ~/ntop
```

2. Create `docker-compose.yml`:

```yaml
version: '3'
services:
  ntopng:
    image: ntop/ntopng:latest
    container_name: ntopng
    network_mode: host
    restart: unless-stopped
    volumes:
      - ./ntopng-data:/data
    command: ["-i", "monitor0", "-w", "3000", "-m", "192.168.1.0/24"]
```

3. Start container:

```bash
sudo docker-compose up -d
```

4. Verify:

```bash
sudo docker ps
```

5. Access UI from browser:

```
http://<VM-IP>:3000
```

- Default credentials: `admin/admin` — **change immediately**  

---

### 7. Verification (Without Cables)

1. Confirm VM sees two interfaces:

```bash
ip link show
```

2. Confirm management interface has IP and monitor interface is UP & promiscuous:

```bash
ip addr
sudo ip link set monitor0 promisc on
```

- The ntopng container is running and ready to capture traffic once WAN/LAN are connected.  

---

## Notes

- Physical cables will be connected later in the lab.  
- Current setup allows full passive monitoring on a single VM.  
- Additional tools (Suricata, Zeek) can be added to the same VM for IDS capabilities.
```

