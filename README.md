# NetworkEdgeTap
an all-in-one network monitoring VM at the edge of a network to passively observe traffic between WAN and LAN, visualize flows, and prepare for IDS integration (Suricata/Zeek).

Inline Network Monitoring Lab — Setup Documentation
Project Goal

Set up an all-in-one network monitoring VM at the edge of a network to passively observe traffic between WAN and LAN, visualize flows, and prepare for IDS integration (Suricata/Zeek).

Hardware Used

Lenovo ThinkCentre M910x Mini PC

Intel I350 4-port Gigabit NIC (used for WAN/LAN and monitoring)

Home/Enterprise router and modem (for traffic source)

Proxmox installed on local storage

Software Used

Proxmox VE (virtualization host)

Ubuntu 24.04.3 LTS VM (base for monitoring)

Docker + Docker Compose for containerized tools

ntopng container for network traffic visualization

Steps Completed
1. Install Proxmox

Download Proxmox VE ISO from Proxmox Download
.

Boot the Mini PC from USB and install Proxmox to the internal storage.

Configure networking for management interface (vmbr0) during installation.

2. Create Ubuntu VM

In Proxmox Web GUI, create a new VM.

Assign minimal resources for monitoring VM:

CPU: 1 core

Memory: 2 GB

Disk: 12 GB

Set OSType: Linux 64-bit and boot from ISO or prebuilt Ubuntu template.

Do not boot the VM yet until network configuration is ready.

3. Configure Host Bridges in Proxmox

vmbr0: Management bridge (Proxmox host + VM SSH access)

vmbr1 (planned): Inline bridge connecting WAN/LAN NICs

Example:
auto vmbr1
iface vmbr1 inet manual
    bridge-ports enp1s0f0 enp1s0f1
    bridge-stp off
    bridge-fd 0


enp1s0f0 → WAN

enp1s0f1 → LAN

The bridge is passive, allows traffic to be mirrored into the VM.

No cables were physically connected at this stage — we only configured logical bridging.

4. Assign Network Interfaces to VM

VM net0: Management interface → connected to vmbr0

VM net1: Monitor interface → connected to vmbr1

net0: virtio=MAC,bridge=vmbr0
net1: virtio=MAC,bridge=vmbr1,promisc=1


Promiscuous mode is enabled manually in the VM after boot:

sudo ip link set monitor0 up
sudo ip link set monitor0 promisc on

5. Install Docker & Docker Compose in VM
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker
sudo systemctl start docker

6. Deploy ntopng Docker Container

Create project folder:

mkdir ~/ntop && cd ~/ntop


Create docker-compose.yml:

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


Start container:

sudo docker-compose up -d


Verify:

sudo docker ps


Access UI from browser:

http://<VM-IP>:3000


Default credentials: admin/admin

Change password immediately

7. Verification (Without Cables)

Confirm VM sees two interfaces:

ip link show


Confirm management interface has IP and monitor interface is UP & promiscuous:

ip addr
sudo ip link set monitor0 promisc on


ntopng container is running and ready to capture traffic once WAN/LAN are connected.

Notes

Physical cables will be connected later in the lab.

Current setup allows full passive monitoring on a single VM.

Additional tools (Suricata, Zeek) can be added to the same VM for IDS capabilities.
