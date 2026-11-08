# Cybersecurity Lab Environment Setup

**Cybersecurity-Lab-Setup-SpicyMeatball41-Week1**

Building an isolated virtual lab for penetration testing and ethical hacking practice.

<img width="1366" height="734" alt="image" src="https://github.com/user-attachments/assets/f694af23-45a1-4f67-aef2-9239d56c812b" />


![Skill](https://img.shields.io/badge/Skill-Cybersecurity-red) ![VirtualBox](https://img.shields.io/badge/VirtualBox-7.2-blue) ![Kali](https://img.shields.io/badge/Kali%20Linux-2026.3-black) ![Network](https://img.shields.io/badge/Network-10.0.0.0%2F24-teal)

---

## Project Overview

This project sets up a virtual cybersecurity and penetration-testing laboratory using VirtualBox and Kali Linux, completed as Week 1 (Project Module 1) of the Networkwalks Cybersecurity program.

The lab runs on a private virtual network so that more machines can be added later and used as targets for **authorized** security testing only. Everything in this lab runs on my own devices.

**Author:** SpicyMeatball41

---

## Objectives

- [x] Use VirtualBox as the virtualization base (latest recommended version)
- [x] Set up Kali Linux as the attacking machine
- [x] Create a NAT Network in the 10.0.0.0/24 subnet
- [x] Assign Kali a static IP of 10.0.0.2/24
- [x] Give Kali full Internet access
- [ ] Enable bidirectional clipboard and drag and drop
- [ ] Share the host Downloads folder with the VM
- [ ] Take a snapshot of the VM

> Tick the last three boxes once you have completed them.

---

## Lab Architecture

| Component | Detail |
|---|---|
| Hypervisor | Oracle VirtualBox 7.2.x |
| Attacker VM | Kali Linux 2026.3 (prebuilt VirtualBox image) |
| Network type | NAT Network (`NatNetwork`) |
| Subnet | 10.0.0.0/24 |
| Kali IP | 10.0.0.2/24 |
| Gateway | 10.0.0.1 |
| DNS | 8.8.8.8 |
| VM memory | 1.5 to 2 GB (host has about 3.3 GB RAM) |

---

## Implementation Steps

### 1. Install VirtualBox

Installed from the Kali repositories on the host:

```bash
sudo apt update
sudo apt install -y virtualbox virtualbox-qt
sudo modprobe vboxdrv
```

### 2. Create the NAT Network

File → Tools → Network → NAT Networks → Create, with:

- Name: `NatNetwork`
- IPv4 Prefix: `10.0.0.0/24`
- DHCP: enabled

Command-line equivalent:

```bash
VBoxManage natnetwork add --netname NatNetwork --network "10.0.0.0/24" --enable --dhcp on
VBoxManage list natnetworks
```

### 3. Download and import Kali Linux

1. Downloaded the prebuilt **Kali Linux VirtualBox (64-bit)** image from kali.org.
2. Extracted it with 7-Zip:
   ```bash
   7z x kali-linux-2026.3-virtualbox-amd64.7z
   ```
3. Imported it with Machine → Add and selected the `.vbox` file.

### 4. Configure the VM settings

- **Network → Adapter 1:** Attached to `NAT Network`, Name `NatNetwork`
- **General → Advanced:** Shared Clipboard = Bidirectional, Drag'n'Drop = Bidirectional
- **Shared Folders:** host Downloads folder added with Auto-mount and Make Permanent enabled

### 5. Configure the static IP in Kali

```bash
sudo nmcli con mod "Wired connection 1" ipv4.method manual \
  ipv4.addresses 10.0.0.2/24 ipv4.gateway 10.0.0.1 \
  ipv4.dns 8.8.8.8 ipv4.dad-timeout 0
sudo nmcli con down "Wired connection 1"
sudo nmcli con up "Wired connection 1"
```

The `ipv4.dad-timeout 0` setting is the known fix for the no-Internet issue seen with VirtualBox 7 and newer Kali releases.

### 6. Shared folder access

```bash
sudo usermod -aG vboxsf $USER
sudo reboot
ls /media/sf_Downloads
```

### 7. Take a snapshot

Machine → Take Snapshot, named after the completed lab setup.

---

## Verification

```bash
ip a
ping -c 3 10.0.0.1
ping -c 3 8.8.8.8
ping -c 3 google.com
```

Results:

- `eth0` shows the static address on the 10.0.0.0/24 network
- Gateway: 3/3 replies, 0% packet loss
- Internet (8.8.8.8): 3/3 replies, 0% packet loss
- DNS (google.com): resolves and replies 3/3, 0% packet loss

---

## Screenshots

| Step | Screenshot |
|---|---|
| NAT Network created in VirtualBox | `screenshots/01-nat-network.png` |
| Kali VM network adapter settings | `screenshots/02-vm-network.png` |
| Clipboard and drag/drop settings | `screenshots/03-clipboard-dragdrop.png` |
| Shared folder settings | `screenshots/04-shared-folders.png` |
| Static IP and ping tests | `screenshots/05-ip-and-pings.png` |
| Snapshot taken | `screenshots/06-snapshot.png` |

Embed each one with `![description](screenshots/file.png)` once you upload them.

---

## Troubleshooting: Problems I Faced and How I Solved Them

### Problem 1: `apt` failed to download packages (TLS error)

**Symptom:**
```
SSL connection failed: error:0A000458:SSL routines::tlsv1 unrecognized name
Unable to fetch some archives
```

**Cause:** My `sources.list` used the `http.kali.org` redirector, which picks a different mirror for each package. One mirror (`kali.darklab.sh`) had a broken TLS configuration, so some packages failed to download and the VirtualBox install aborted.

**Solution:** I pinned apt to a single working mirror, then refreshed and reinstalled:

```bash
sudo sed -i 's#https://http.kali.org/kali#https://kali.download/kali#g' /etc/apt/sources.list
sudo apt clean
sudo apt update
sudo apt install -y virtualbox virtualbox-qt
```

### Problem 2: `No space left on device`

**Symptom:** VirtualBox reported `errno=28 : No space left on device`.

**Cause:** The 116 GB root partition was 100% full (about 115 MB free), and everything lived on that one partition.

**Solution:** I used `df -h` and `du` to find the largest items:

```bash
df -h
sudo du -xh / --max-depth=1 2>/dev/null | sort -rh | head
sudo find / -xdev -type f -size +1G -exec ls -lh {} \; 2>/dev/null
```

The space was mostly old local AI models, caches, the Trash and a large wordlist collection. I cleared caches and Trash, removed models I no longer needed using the proper tool, deleted the extra archive, and freed enough space for the VM (about 15 GB or more).

### Problem 3: Could not remove the swap file

**Symptom:** `swapoff /swapfile` was killed and `rm` said `Operation not permitted`.

**Cause:** The host has only about 3.3 GB of RAM, so the system could not move the swapped data back into memory, and the out-of-memory killer stopped `swapoff`. Because swap was still active, the file could not be removed.

**Solution:** I stopped using the swap file at boot (commented the entry in `/etc/fstab`), rebooted, and removed or resized it while it was unused.

### Problem 4: VM would not boot ("failed to boot / missing operating system")

**Symptom:** VirtualBox showed a boot failure message and the VM sat at the GRUB screen.

**Cause:** Possibly an incomplete extraction (the disk had filled up earlier) or a boot or firmware setting mismatch.

**Solution:** I checked that the `.vdi` was attached under Settings → Storage, that Hard Disk was first in the boot order, toggled **Enable EFI**, and verified the archive and disk image with `7z t` and `VBoxManage showmediuminfo`. *(Edit this line to state exactly which step fixed it for you.)*

### Problem 5: Low RAM on the host

**Symptom:** The host was slow and swapping heavily while the VM ran.

**Solution:** I reduced the VM's memory to 1.5 to 2 GB, stopped heavy background services before starting it, and closed other large applications.

---

## Key Learnings

- How to build an isolated NAT Network in VirtualBox with a custom subnet
- Configuring a static IP with `nmcli` and understanding gateway and DNS settings
- Diagnosing apt mirror problems, disk-space exhaustion and low-memory behaviour on Linux
- Verifying connectivity layer by layer (gateway, then Internet by IP, then DNS)
- Using snapshots to keep a known-good baseline for future labs

---

## Legal and Ethical Notice

This lab is for education and research only. It runs on my own devices in an isolated virtual environment. Hacking is only legal when I test systems I own, have written permission to test, or am working under a signed agreement with an agreed scope.

---

## Next Steps

- Add more virtual machines (Windows and Android) to the same NAT Network for ping tests and practice targets
- Take snapshots of each machine before testing
