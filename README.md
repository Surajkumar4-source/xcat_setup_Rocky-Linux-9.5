# 🎓 COMPLETE xCAT SETUP GUIDE 

## Rocky Linux 9.5 — Management Node + Compute Node (Diskful Install)
### 📘 For Beginners — Copy-Paste Ready — All Known Issues Pre-Fixed

---

## 📋 BEFORE YOU START — Read This

This guide is **battle-tested** — every command here already works, because we hit these exact errors and fixed them. Follow **in exact order**, don't skip steps.

### 🖥️ What You Need

| Item | Spec |
|------|------|
| **Management Node VM** | Rocky Linux 9.5, 4GB+ RAM, 40GB+ disk, 2 NICs |
| **Compute Node VM** | Empty (no OS), 2GB RAM, 20GB disk, 1 NIC, BIOS firmware |
| **Rocky Linux 9.5 DVD ISO** | Downloaded on your host PC beforehand |
| **VMware Network** | 1 Bridged/NAT (internet) + 1 Host-only (cluster) |

### 🧭 Network Plan (adjust IPs to match YOUR VMware host-only subnet)

```
Management Node (xcatmn):
  NIC1 (internet)     → DHCP/NAT/Bridged  → e.g. 192.168.197.129
  NIC2 (cluster)      → STATIC            → 192.168.78.129

Compute Node (compute01):
  NIC1 (cluster only) → gets IP from xCAT DHCP → 192.168.78.10
```

> ⚠️ **Check your own VMware "Host-only" subnet first** (Edit → Virtual Network Editor) and replace `192.168.78.x` below with YOUR actual subnet if different.

---

# 🔹 PHASE 0 — VMware VM Preparation (Do This BEFORE Installing OS)

## Management Node VM Settings
```
Name: xcatmn
RAM: 4096 MB
CPU: 2 cores
Disk: 40GB+
Network Adapter 1: NAT or Bridged (internet)
Network Adapter 2: Host-only (Add via "Add Device")
```

## Compute Node VM Settings (create now, install nothing!)
```
Name: compute01
RAM: 2048 MB
CPU: 1 core
Disk: 20GB (empty, new)
Network Adapter: Host-only ONLY (must match MN's NIC2 subnet)
CD/DVD: Disconnected / None
⚠️ CRITICAL: Options tab → Advanced → Firmware type = BIOS (NOT UEFI)
```

**Get compute01's MAC address now** (VM powered OFF):
```
compute01 Settings → Network Adapter → Advanced button → copy MAC Address
Write it down! Example: 00:50:56:2D:7D:7A
```

---

# 🔹 PHASE 1 — Base OS Setup (On Management Node)

Install Rocky Linux 9.5 on the MN VM normally, then run:

```bash
# ---- Step 1: Set hostname ----
hostnamectl set-hostname xcatmn.cluster.local
exec bash

# ---- Step 2: Check your NICs ----
ip -br -c a
```

📝 **Note down which NIC is which** (internet vs host-only) — you'll need exact names below.

```bash
# ---- Step 3: Set STATIC IP on the host-only/cluster NIC ----
# Replace "ens192" with YOUR cluster NIC name, and IP with YOUR subnet
nmcli con mod ens192 ipv4.addresses 192.168.78.129/24
nmcli con mod ens192 ipv4.method manual
nmcli con mod ens192 connection.autoconnect yes
nmcli con up ens192

# ---- Step 4: Verify ----
ip -br -c a
```

```bash
# ---- Step 5: Disable firewall & SELinux (LAB ONLY) ----
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```

```bash
# ---- Step 6: Update /etc/hosts ----
cat >> /etc/hosts << 'EOF'
192.168.78.129   xcatmn.cluster.local   xcatmn
192.168.78.10    compute01.cluster.local compute01
EOF
```

```bash
# ---- Step 7: Update system + basic tools ----
dnf update -y
dnf install -y wget vim net-tools bind-utils tar bzip2 perl screen
```

---

# 🔹 PHASE 2 — Install xCAT (Repos Fixed)

```bash
# ---- Step 1: xcat-core repo (using "latest" NOT "devel"!) ----
cat > /etc/yum.repos.d/xcat-core.repo << 'EOF'
[xcat-core]
name=xCAT 2 Core packages
baseurl=https://xcat.org/files/xcat/repos/yum/latest/xcat-core
enabled=1
gpgcheck=1
gpgkey=https://xcat.org/files/xcat/repos/yum/latest/xcat-core/repodata/repomd.xml.key
EOF
```

```bash
# ---- Step 2: xcat-dep repo ----
wget -O /etc/yum.repos.d/xcat-dep.repo \
  https://xcat.org/files/xcat/repos/yum/xcat-dep/rh9/x86_64/xcat-dep.repo
```

```bash
# ---- Step 3: Enable CRB + EPEL (correct order!) ----
dnf install -y dnf-plugins-core
dnf config-manager --set-enabled crb
dnf install -y epel-release
```

```bash
# ---- Step 4: Verify all repos ----
dnf repolist
```
✅ You must see: `xcat-core`, `xcat-dep`, `crb`, `epel`, `baseos`, `appstream`

```bash
# ---- Step 5: Install xCAT ----
dnf install -y xCAT
```

```bash
# ---- Step 6: Load xCAT environment ----
source /etc/profile.d/xcat.sh
echo "source /etc/profile.d/xcat.sh" >> ~/.bashrc
```

```bash
# ---- Step 7: Verify xCAT is running ----
systemctl status xcatd
lsdef -t site
```
✅ Should show `active (running)` and a site table dump.

---

# 🔹 PHASE 3 — Configure Network Services (DNS/DHCP/TFTP)

```bash
# ---- Step 1: Site table settings ----
# Replace ens192 and 192.168.78.129 with YOUR values
chdef -t site master=192.168.78.129
chdef -t site domain=cluster.local
chdef -t site forwarders=8.8.8.8,8.8.4.4
chdef -t site nameservers=192.168.78.129
chdef -t site dhcpinterfaces=ens192
```

```bash
# ---- Step 2: Detect network ----
makenetworks
lsdef -t network
```
📝 Note the exact network object name shown (e.g. `192_168_78_0-255_255_255_0`)

```bash
# ---- Step 3: Configure the cluster network (with dynamic range!) ----
chdef -t network 192_168_78_0-255_255_255_0 \
  net=192.168.78.0 \
  mask=255.255.255.0 \
  gateway=192.168.78.129 \
  tftpserver=192.168.78.129 \
  nameservers=192.168.78.129 \
  dhcpserver=192.168.78.129 \
  dynamicrange=192.168.78.200-192.168.78.250
```
> 💡 `dynamicrange` prevents the "no dynamic range" DHCP warning.

```bash
# ---- Step 4: DNS setup ----
# -4 flag avoids IPv6 resolution errors/noise in logs
echo 'OPTIONS="-4"' >> /etc/sysconfig/named
makedns -n
systemctl enable named
systemctl restart named
systemctl status named
```

```bash
# ---- Step 5: DHCP setup ----
makedhcp -n
makedhcp -a
systemctl enable dhcpd
systemctl restart dhcpd
systemctl status dhcpd
```

```bash
# ---- Step 6: TFTP — CRITICAL FIX ----
# xCAT runs its OWN TFTP (in.tftpd) via xcatd — the systemd tftp.socket
# service WILL conflict on port 69. We must mask it permanently.
systemctl disable tftp.socket 2>/dev/null
systemctl stop tftp.socket 2>/dev/null
systemctl mask tftp.socket
systemctl mask tftp.service
systemctl reset-failed tftp.socket 2>/dev/null

# Verify xCAT's own TFTP is running (started automatically by xcatd)
ps aux | grep in.tftpd | grep -v grep
```
✅ You should see `/usr/sbin/in.tftpd ... -s /tftpboot` running as part of xcatd.

```bash
# ---- Step 7: Full Phase 3 verification ----
echo "--- DNS ---" && systemctl is-active named
echo "--- DHCP ---" && systemctl is-active dhcpd
echo "--- TFTP process ---" && ps aux | grep -v grep | grep in.tftpd
echo "--- TFTP port ---" && ss -ulnp | grep :69
echo "--- DNS resolve test ---" && nslookup xcatmn 192.168.78.129
```

---

# 🔹 PHASE 4 — Import Rocky Linux 9.5 ISO

```bash
# ---- Step 1: Check disk space FIRST ----
df -h /
```
⚠️ You need **at least 15GB free** on `/`. If not, free up space before continuing.

```bash
# ---- Step 2: Copy ISO to Management Node ----
# Option A - from your Windows PC via WinSCP/MobaXterm drag-drop to /root/
# Option B - via scp from your host:
#   scp Rocky-9.5-x86_64-dvd.iso root@192.168.78.129:/root/
#
# ⚠️ IMPORTANT: Put ISO in /root/ (NOT /home/) — avoids separate /home
# partition running out of space, which we hit during testing!
```

```bash
# ---- Step 3: Verify ISO integrity (optional but recommended) ----
sha256sum /root/Rocky-9.5-x86_64-dvd.iso
# Compare with: https://download.rockylinux.org/pub/rocky/9/isos/x86_64/CHECKSUM
```

```bash
# ---- Step 4: Import ISO into xCAT (use screen to survive SSH drops) ----
screen -S copycds
copycds /root/Rocky-9.5-x86_64-dvd.iso
```
⏳ Takes 5–15 minutes. Wait for: `Media copy operation successful`
Press `Ctrl+A` then `D` to detach from screen if needed; reattach with `screen -r copycds`.

```bash
# ---- Step 5: Free up space — delete ISO (already imported!) ----
rm -f /root/Rocky-9.5-x86_64-dvd.iso
df -h /
```

```bash
# ---- Step 6: Verify OS images created ----
lsdef -t osimage
```
✅ Must show `rocky9.5-x86_64-install-compute` (this is what we use for diskful install)

```bash
# ---- Step 7: CRITICAL FIX — Apache 403 error on /install ----
# xCAT's default xcat.conf is missing "Indexes" option, causing HTTP 403
sed -i 's|Options FollowSymLinks Includes MultiViews|Options Indexes FollowSymLinks Includes MultiViews|g' \
  /etc/httpd/conf.d/xcat.conf

systemctl restart httpd
```

```bash
# ---- Step 8: Verify HTTP serves packages correctly ----
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" \
  http://192.168.78.129/install/rocky9.5/x86_64/
```
✅ Must show `HTTP Status: 200` (NOT 403!)

---

# 🔹 PHASE 5 — Define the Compute Node

```bash
# ---- Step 1: Define compute01 (use YOUR MAC address from Phase 0!) ----
mkdef -t node compute01 \
  groups=compute,all \
  arch=x86_64 \
  mac=00:50:56:2D:7D:7A \
  ip=192.168.78.10 \
  netboot=xnba \
  installnic=mac \
  primarynic=mac \
  mgt=ipmi

# ---- Step 2: Verify ----
lsdef compute01
```

```bash
# ---- Step 3: Update DNS/hosts/DHCP for the new node ----
makehosts compute01
makedns -n
makedhcp compute01
```

```bash
# ---- Step 4: Verify DNS + DHCP entries ----
nslookup compute01 192.168.78.129
grep -A4 -i "50:56:2d" /etc/dhcp/dhcpd.conf
```

```bash
# ---- Step 5: Set root password for installed node ----
chtab key=system passwd.username=root passwd.password=cluster@123
```

```bash
# ---- Step 6: Assign osimage & prepare PXE boot ----
nodeset compute01 osimage=rocky9.5-x86_64-install-compute
```
✅ Expected: `compute01: install rocky9.5-x86_64-compute`

```bash
# ---- Step 7: Verify PXE + kickstart files exist ----
ls -la /tftpboot/xcat/xnba/nodes/
ls -la /install/autoinst/compute01
```

---

# 🔹 PHASE 6 — Boot the Compute Node!

## Terminal 1 (on MN) — Watch DHCP
```bash
journalctl -u dhcpd -f
```

## Terminal 2 (on MN) — Watch xCAT log
```bash
tail -f /var/log/xcat/xcat.log
```

## Terminal 3 (on MN) — Watch node status
```bash
watch -n 10 nodestat compute01
```

## Now — Power ON compute01 in VMware
```
VMware → compute01 → Power On
Watch console: PXE boot → DHCP → downloads installer → Anaconda installs Rocky 9.5 automatically
```

⏳ **15–30 minutes total.** Status flow: `booting` → `installing` → `booted`

---

# 🔹 PHASE 7 — Post-Install: Fix SSH the PROPER xCAT Way

> ⚠️ **Do NOT manually edit `/etc/ssh/sshd_config` on the compute node!**
> xCAT has a built-in, correct way to establish passwordless SSH trust.

```bash
# ---- Step 1: Push MN's SSH key to compute01 (one-time password entry) ----
xdsh compute01 -K
```
It will prompt for compute01's root password (`cluster@123` from Phase 5) **once**.

```bash
# ---- Step 2: Test passwordless access ----
ssh compute01 hostname
```
✅ Should log in with **NO password prompt**.

```bash
# ---- Step 3: Run parallel commands (the real power of xCAT!) ----
xdsh compute01 "hostname && uname -r"
xdsh compute01 "cat /etc/os-release | grep PRETTY_NAME"
xdsh compute01 "df -h /"
```

---

# ✅ FINAL VERIFICATION — Everything Working

```bash
echo "=== xCAT Services ==="
systemctl is-active named dhcpd xcatd httpd

echo "=== Node Definition ==="
lsdef compute01 -i mac,ip,netboot

echo "=== Node Status ==="
nodestat compute01

echo "=== SSH/xdsh Test ==="
xdsh compute01 "hostname && uname -r"

echo "=== Compute OS Check ==="
ssh compute01 "cat /etc/os-release | head -4"
```

---


# 🗺️ Complete Command Cheat Sheet


## 1. Node Management Commands

| Command    | Purpose                                                                                             | Example                                               |
| ---------- | --------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| `lsdef`    | Lists the definition (configuration) of one or more nodes, groups, networks, or other xCAT objects. | `lsdef compute01`                                     |
| `mkdef`    | Creates a new node or object definition in the xCAT database.                                       | `mkdef -t node compute01 groups=compute ip=10.1.1.10` |
| `chdef`    | Modifies an existing node or object definition.                                                     | `chdef compute01 netboot=xnba`                        |
| `rmdef`    | Deletes a node or object definition from xCAT.                                                      | `rmdef compute01`                                     |
| `nodels`   | Displays node names and their attributes.                                                           | `nodels all nodelist.groups`                          |
| `nodeadd`  | Adds new nodes to an existing node group.                                                           | `nodeadd compute01 compute`                           |
| `nodech`   | Changes node group membership or selected attributes.                                               | `nodech compute01 groups=service`                     |
| `noderm`   | Removes nodes from a node group.                                                                    | `noderm compute01 compute`                            |
| `nodestat` | Displays the current state of nodes (booting, installing, running, etc.).                           | `nodestat compute01`                                  |

---

## 2. Power Management

| Command     | Purpose                                                                                   | Example                         |
| ----------- | ----------------------------------------------------------------------------------------- | ------------------------------- |
| `rpower`    | Turns systems on, off, resets them, or queries their power status using the BMC.          | `rpower compute01 stat`         |
| `rspconfig` | Configures service processor/BMC settings such as network configuration or users.         | `rspconfig compute01`           |
| `renergy`   | Collects power and energy consumption data from supported hardware.                       | `renergy compute01`             |
| `rvitals`   | Displays hardware health information such as temperatures, voltages, fans, and CPU usage. | `rvitals compute01`             |
| `rbeacon`   | Turns the chassis identify LED on or off to locate a server in a rack.                    | `rbeacon compute01 on`          |
| `rflash`    | Updates firmware on servers or service processors.                                        | `rflash compute01 firmware.img` |

---

## 3. Remote Access

| Command | Purpose                                                   | Example                              |
| ------- | --------------------------------------------------------- | ------------------------------------ |
| `xdsh`  | Executes shell commands on multiple nodes simultaneously. | `xdsh compute "hostname"`            |
| `psh`   | Parallel shell command execution on multiple systems.     | `psh compute uptime`                 |
| `xdcp`  | Copies files to many nodes simultaneously.                | `xdcp compute /etc/hosts /etc/hosts` |
| `pcp`   | Parallel copy utility similar to `xdcp`.                  | `pcp file.txt compute:/tmp`          |
| `rcons` | Opens the remote console of a node through the BMC.       | `rcons compute01`                    |

---

## 4. Provisioning Commands

| Command      | Purpose                                                                    | Example                     |
| ------------ | -------------------------------------------------------------------------- | --------------------------- |
| `nodeset`    | Sets the next boot action (install, network boot, shell, etc.).            | `nodeset compute01 install` |
| `rinstall`   | Starts an operating system installation on selected nodes.                 | `rinstall compute01`        |
| `updatenode` | Installs software, synchronizes files, or updates configurations on nodes. | `updatenode compute01 -F`   |
| `genimage`   | Builds a stateless operating system image.                                 | `genimage centos8`          |
| `packimage`  | Packages an OS image for deployment.                                       | `packimage centos8`         |
| `liteimg`    | Creates a statelite image.                                                 | `liteimg centos8`           |
| `mknetboot`  | Generates network boot files.                                              | `mknetboot compute01`       |
| `copycds`    | Imports Linux installation media into xCAT.                                | `copycds /mnt/iso`          |
| `imgcapture` | Captures the operating system from a running node into a deployable image. | `imgcapture compute01`      |

---

## 5. Network Management

| Command          | Purpose                                            | Example          |
| ---------------- | -------------------------------------------------- | ---------------- |
| `makedhcp`       | Creates or updates the DHCP configuration.         | `makedhcp -n`    |
| `makedns`        | Generates DNS entries for managed nodes.           | `makedns -n`     |
| `makehosts`      | Generates the `/etc/hosts` file.                   | `makehosts`      |
| `makenetworks`   | Creates network configuration entries.             | `makenetworks`   |
| `makeknownhosts` | Generates SSH `known_hosts` entries for all nodes. | `makeknownhosts` |
| `makentp`        | Configures NTP settings on nodes.                  | `makentp`        |

---

## 6. Inventory and Monitoring

| Command   | Purpose                                                                                | Example              |
| --------- | -------------------------------------------------------------------------------------- | -------------------- |
| `rinv`    | Retrieves hardware inventory such as CPU, memory, disks, serial numbers, and adapters. | `rinv compute01 all` |
| `monadd`  | Adds monitoring plugins.                                                               | `monadd ganglia`     |
| `monls`   | Lists configured monitoring plugins.                                                   | `monls`              |
| `monrm`   | Removes monitoring plugins.                                                            | `monrm ganglia`      |
| `rvitals` | Retrieves live hardware sensor information.                                            | `rvitals compute01`  |

---

## 7. Database Management

| Command         | Purpose                                         | Example                    |
| --------------- | ----------------------------------------------- | -------------------------- |
| `chtab`         | Modifies records in xCAT database tables.       | `chtab key=value`          |
| `gettab`        | Displays records from xCAT database tables.     | `gettab hosts`             |
| `tabdump`       | Exports xCAT database tables to text files.     | `tabdump site`             |
| `tabrestore`    | Restores database tables from exported files.   | `tabrestore site`          |
| `dumpxCATdb`    | Creates a complete backup of the xCAT database. | `dumpxCATdb`               |
| `restorexCATdb` | Restores the xCAT database from a backup.       | `restorexCATdb backup.sql` |

---

## 8. Console & Boot

| Command           | Purpose                                             | Example                  |
| ----------------- | --------------------------------------------------- | ------------------------ |
| `rcons`           | Opens the server console.                           | `rcons compute01`        |
| `makeconservercf` | Generates the Conserver configuration file.         | `makeconservercf`        |
| `rsetboot`        | Changes the next boot device (disk, PXE, CD, etc.). | `rsetboot compute01 net` |
| `bootseq`         | Displays or modifies the BIOS boot order.           | `bootseq compute01`      |

---

These are the commands that administrators use most frequently in day-to-day xCAT management.















<br>

<br>
<br>






### Unified Dashboard Snapshots 
---
<br>
<br>

## 1. ====== 📊 Dashboard Overview ====== 
---


<img width="1853" height="1023" alt="Screenshot 2026-08-19 110819" src="https://github.com/user-attachments/assets/77dcc413-ed80-4698-a0b1-5f5e9c1fdd77" />



<br>


<img width="1853" height="1020" alt="Screenshot 2026-08-19 110848" src="https://github.com/user-attachments/assets/053d0dfd-581e-4ebb-b899-91079c0b4f1e" />



<br>
<br>

## 2. ======  💓 Health Check ======
---

<br>


<img width="1844" height="1015" alt="Screenshot 2026-08-19 111117" src="https://github.com/user-attachments/assets/d345b8b2-7ca6-43fd-bfe6-a1895fb26c06" />



<br>
<br>

## 3. ====== 🖥️ Node List ======  
---

<br>

<img width="1850" height="1015" alt="Screenshot 2026-08-19 111342" src="https://github.com/user-attachments/assets/89ee15a0-48b8-40a9-8dce-0adec9bd6e93" />



<br>
<br>

## 4. ====== ➕ Add Node ======  
---
<br>


<img width="1848" height="1019" alt="Screenshot 2026-08-19 111525" src="https://github.com/user-attachments/assets/512ce34c-4a8d-4a46-97a2-d6546e87ecfe" />




<br>
<br>


## 5. ====== 📦 Bulk Add Nodes ======  
---
<br>



<img width="1853" height="1006" alt="Screenshot 2026-08-19 111544" src="https://github.com/user-attachments/assets/8ab14199-c20c-42a0-903e-af556f172d7c" />



<br>
<br>



## 6. ====== 🚀 Provision ======  
---
<br>


<img width="1845" height="1016" alt="Screenshot 2026-08-19 111930" src="https://github.com/user-attachments/assets/36a862a3-737f-4fbc-887a-7ec41836168e" />



<br>
<br>



## 7. ====== 💻 Remote Command ======  
---
<br>


<img width="1852" height="1017" alt="Screenshot 2026-08-19 111947" src="https://github.com/user-attachments/assets/b09e5b20-5477-43d2-96d9-5a29ff021d31" />



<br>
<br>



## 8. ====== 🔑 SSH Key Setup ======  
---
<br>


<img width="1852" height="1018" alt="Screenshot 2026-08-19 112008" src="https://github.com/user-attachments/assets/84dc5d9d-586d-49c0-97c4-10bc12bee8ca" />



<br>
<br>



## 9. ====== ⚙️ Services ======  
---
<br>

<img width="1849" height="1012" alt="Screenshot 2026-08-19 112256" src="https://github.com/user-attachments/assets/07847111-b624-403e-8a31-edc8044c2aff" />



<br>
<br>



## 10. ====== 💿 OS Images ======  
---
<br>


<img width="1846" height="1012" alt="Screenshot 2026-08-19 112317" src="https://github.com/user-attachments/assets/b28b0624-bf4c-4804-8af7-e9bafffd5abf" />



<br>
<br>



## 11. ====== 🌐 Networks ======
---

<br>


<img width="1841" height="1017" alt="Screenshot 2026-08-19 112342" src="https://github.com/user-attachments/assets/14cf1319-1df3-406a-ba99-49ce531c93e8" />


<br>
<br>






## 12. ====== 🔄 Rebuild Services ======
---

<br>


<img width="1855" height="1005" alt="Screenshot 2026-08-19 112410" src="https://github.com/user-attachments/assets/0bdc8556-17ae-40c5-a903-695726298876" />


<br>
<br>





## 13. ====== 📜 Logs ======
---

<br>

<img width="1846" height="1020" alt="Screenshot 2026-08-19 112446" src="https://github.com/user-attachments/assets/4f630c8b-b47b-46aa-b403-85e345abe0c9" />



<br>
<br>






## 14. ====== 🗄️ xCAT Tables ======
---

<br>

<img width="1842" height="1016" alt="Screenshot 2026-08-19 112505" src="https://github.com/user-attachments/assets/fd00fa13-cdba-4a5b-9864-69ff266f6cce" />



<br>
<br>





## 15. ====== 🎬 Live Boot Console ======
---

<br>

<img width="1845" height="1009" alt="Screenshot 2026-08-19 112606" src="https://github.com/user-attachments/assets/ccd9fb31-588d-4795-b1f6-437f1e725526" />



<br>
<br>



## 16. ====== 🏗️ Cluster Architecture ======
---

<br>


<img width="1848" height="1020" alt="Screenshot 2026-08-19 112732" src="https://github.com/user-attachments/assets/72fb08d4-7d82-4345-be02-57ecbad2508e" />


<br>
<br>


