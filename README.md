# -NETWORKWALKS-B084C-WK1-PM1-CYBERSECURITY-LAB-SETUP-SAMPLE
LAB1
# Kali Linux on VirtualBox: NAT Network Setup & Troubleshooting

**Author:** Arlick
**Date:** 30 October 2026
**Tools:** Oracle VirtualBox, Kali Linux 2026.2, NetworkManager (`nmcli`)

## Objective

Give a Kali Linux VM a static IP address on a VirtualBox **NAT Network** (`10.0.0.0/24`) so it can reach the internet and share a lab network with the other VMs (Windows Server, Windows 11, etc.).

## Lab Network Plan

| Item            | Value          |
|-----------------|----------------|
| NAT Network     | `NatNetwork`   |
| Subnet          | `10.0.0.0/24`  |
| Gateway         | `10.0.0.1`     |
| Kali IP address | `10.0.0.2/24`  |
| DNS             | `8.8.8.8` (fallback: `10.0.0.1`) |

---

## Step 1: Create the NAT Network in VirtualBox

**File → Tools → Network Manager → NAT Networks → Create**

- Name: `NatNetwork`
- IPv4 Prefix: `10.0.0.0/24`
- Enable DHCP: ✔

![Kali static IPv4 settings](Screenshot%202026-10-30%20123133.png)

## Step 2: Attach the Kali VM to the NAT Network

With the VM **powered off**: **select Kali linux → Settings → Network → Adapter 1**

- Enable Network Adapter: ✔
- Attached to: **NAT Network**
- Name: **NatNetwork**
- Promiscuous Mode: `Deny`
- Cable Connected: ✔

Click **OK**.

![Attaching Adapter 1 to NatNetwork](images/02b-attach-adapter-natnetwork.png)

> ⚠️ "NAT" and "NAT Network" are different options in the *Attached to* list. Choose **NAT Network**. Picking plain NAT caused the problem in Step 4.

## Step 3: Set a Static IPv4 Address in Kali

In Kali: **network icon → Edit Connections → Wired connection 1 → IPv4 Settings**

- Method: `Manual`
- Address: `10.0.0.2` · Netmask: `24` · Gateway: `10.0.0.1`
- DNS servers: `8.8.8.8`

![Kali static IPv4 settings](images/03-kali-static-ipv4.png)

Apply the change and restart the connection:

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.dad-timeout 0
sudo nmcli connection down "Wired connection 1"
sudo nmcli connection up "Wired connection 1"
```

![Restarting the connection with nmcli](images/04-nmcli-restart.png)

## Step 4: Problem: No Internet

Testing connectivity failed:

```bash
ping 8.8.8.8
# From 10.0.0.2 icmp_seq=1 Destination Host Unreachable
# 100% packet loss
```

![Ping failing](images/05-ping-unreachable.png)

### Root cause

The VM's network adapter was attached to **NAT**, not **NAT Network**.

![Adapter attached to NAT (wrong)](images/01-adapter-wrong-nat.png)

These are two different VirtualBox modes:

| Mode        | Subnet         | Gateway     | Shared with other VMs? |
|-------------|----------------|-------------|------------------------|
| NAT         | `10.0.2.0/24`  | `10.0.2.2`  | No (each VM is isolated) |
| NAT Network | `10.0.0.0/24`  | `10.0.0.1`  | Yes |

Kali was set up for `10.0.0.0/24`, but the adapter put it on the isolated NAT network, where no gateway at `10.0.0.1` exists. Kali couldn't reach its gateway, so every packet failed with *Destination Host Unreachable*.

## Step 5: Fix

Step 2 had been missed, so the adapter was still on plain NAT. To fix it:

1. Shut down the Kali VM (fully powered off, not saved state).
2. **Settings → Network → Adapter 1**
   - Attached to: **NAT Network**
   - Name: **NatNetwork**
3. Click OK and start the VM.

![Adapter attached to NAT Network (fixed)](images/06-adapter-fixed-natnetwork.png)

## Step 6: Verify

```bash
ip a                 # confirm 10.0.0.2/24 on eth0
ping -c 4 10.0.0.1   # gateway
ping -c 4 8.8.8.8    # internet
ping -c 4 google.com # DNS
```

<!-- Add a screenshot of the successful pings here: -->
<!-- ![Successful ping](images/07-ping-success.png) -->

If the IP works but domain names don't resolve, change DNS to `10.0.0.1`.
If `10.0.0.2` still has issues, try another address such as `10.0.0.10`.

## Step 7: Snapshot the Working State

With the VM powered off: **select Kali linux → Snapshots tab → Take**, then name it `Clean install – network working`.

For a full backup that doesn't depend on the VM's own files, use **Clone → Full clone** or **File → Export Appliance (.ova)**.

---

## Lessons Learned

- **NAT ≠ NAT Network.** Check the adapter mode first when a static IP can't reach its gateway.
- *Destination Host Unreachable* from your own IP means the problem is local (the gateway can't be reached at layer 2), not DNS or the internet.
- Adapter settings can only be changed while the VM is powered off.
- Take a snapshot once the lab works, so you can roll back after experiments.



