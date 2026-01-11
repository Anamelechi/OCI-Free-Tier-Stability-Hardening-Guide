# **OCI Free Tier Stability Hardening Guide**  
### *Stabilizing AMD E2.1.Micro Instances Without Migration*

---

## 🏷️ **Badges**

<p align="left">
  <img src="https://img.shields.io/badge/Cloud-Oracle_Cloud_Infra-red?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Linux-Ubuntu_22.04-orange?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Category-DevOps-blue?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Focus-Reliability_Engineering-green?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Status-Production_Ready-brightgreen?style=for-the-badge" />
</p>

---

# 📘 **Table of Contents**

- [Overview](#overview)  
- [Symptoms](#symptoms)  
- [Root Cause](#root-cause)  
- [Architecture Overview](#architecture-overview)  
- [System Diagram](#system-diagram)  
- [Hardening Steps](#hardening-steps)  
- [Monitoring](#monitoring)  
- [Conclusion](#conclusion)

---

# 🧭 **Overview**

Oracle Cloud’s Always Free AMD E2.1.Micro instances are known to freeze periodically due to **hypervisor instability**.  
This guide provides a **non‑migration hardening strategy** that keeps the instance stable, self‑healing, and resilient.

This is ideal for:
- Cloud engineers  
- DevOps portfolios  
- Lightweight production workloads  
- OCI Free Tier users  

---

# 🚨 **Symptoms**

Common signs of OCI host instability:

- VM becomes unreachable (SSH timeout)  
- Only a **Force Reboot** restores functionality  
- `last -x` shows **reboots without shutdown entries**  
- `system.journal` reports **unclean shutdown**  
- No OOM, no kernel panic, no filesystem errors  

---

# 🧠 **Root Cause**

After analyzing kernel logs and system history:

- The VM is experiencing **cold boots**, not graceful reboots  
- No OS-level crash indicators  
- Disk and network devices reinitialize as if power was cut  
- Uptime patterns (7–25 days) match OCI free-tier host resets  

**Conclusion:**  
Your OS is stable.  
The **OCI hypervisor** is periodically crashing or resetting.

---

# 🏗️ **Architecture Overview**

This hardening strategy introduces:

- **Swap Layer** → Prevents memory stalls  
- **Kernel Auto-Recovery** → Reboots on panic/oops  
- **Hardware Watchdog** → Reboots on scheduler freeze  
- **Network Watchdog** → Reboots on network isolation  
- **Service Optimization** → Reduces load on constrained hardware  

Together, these components create a **self-healing compute instance** that survives OCI host instability.

---

# 🖼️ **System Diagram**

```
                   ┌──────────────────────────────┐
                   │      Oracle Cloud Host        │
                   │  (Unstable Hypervisor Layer)  │
                   └───────────────┬──────────────┘
                                   │
                                   ▼
                     ┌────────────────────────┐
                     │   Ubuntu 22.04 VM      │
                     │  AMD E2.1.Micro Shape  │
                     └──────────────┬─────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        ▼                           ▼                           ▼
┌──────────────┐          ┌────────────────┐          ┌──────────────────┐
│ Swap Layer   │          │ Kernel Auto    │          │ Hardware Watchdog │
│ (1–2 GB)     │          │ Recovery       │          │ (/dev/watchdog)   │
└──────────────┘          └────────────────┘          └──────────────────┘
        │                           │                           │
        └──────────────┬────────────┴──────────────┬────────────┘
                       ▼                           ▼
             ┌────────────────┐          ┌──────────────────────┐
             │ Network         │          │ Service Optimization │
             │ Heartbeat       │          │ (snapd, multipathd) │
             └────────────────┘          └──────────────────────┘
                       │
                       ▼
             ┌────────────────────┐
             │ Self-Healing VM    │
             │ Auto-Reboots on    │
             │ Freeze/Isolation   │
             └────────────────────┘
```

---

# 🔧 **Hardening Steps**

## **1. Add Swap**
```bash
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## **2. Enable Kernel Auto-Recovery**
```bash
echo 'kernel.panic=10' | sudo tee -a /etc/sysctl.conf
echo 'kernel.panic_on_oops=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

## **3. Install Linux Watchdog**
```bash
sudo apt install watchdog
sudo systemctl enable watchdog
sudo systemctl start watchdog
```

Edit config:

```
watchdog-device = /dev/watchdog
interval = 10
max-load-1 = 24
```

---

## **4. Add Network Heartbeat Watchdog**

Create script:

```bash
sudo nano /usr/local/bin/net-watchdog.sh
```

Paste:

```bash
#!/bin/bash
TARGET=8.8.8.8
COUNT=3

if ! ping -c $COUNT $TARGET > /dev/null 2>&1; then
    logger "net-watchdog: network unreachable, rebooting"
    /sbin/reboot -f
fi
```

Cron job:

```
*/5 * * * * /usr/local/bin/net-watchdog.sh
```

---

## **5. Disable Unnecessary Services**
```bash
sudo systemctl disable --now snapd
sudo systemctl disable --now multipathd
sudo systemctl disable --now lvm2-monitor
```

---

## **6. Optional: Recreate Instance on Same Shape**
Not a migration — just a rebuild on the same AMD E2.1.Micro shape.

---

# 📊 **Monitoring**

### Check for unclean shutdowns:
```bash
last -x | head
```

### Check kernel logs:
```bash
sudo dmesg -T | grep -i -E "panic|error|fail|warn|oom|reset"
```

### Check watchdog:
```bash
systemctl status watchdog
```

---

# 🏁 **Conclusion**

This guide transforms an unstable OCI Free Tier instance into a **self-healing, production-ready compute node** that withstands hypervisor instability without manual intervention.

It demonstrates:
- Cloud reliability engineering  
- Linux hardening  
- Real-world troubleshooting  
- Portfolio-grade documentation  
