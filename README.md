# Armbian on MXQ Pro 4K DTV(RK3228A/RK3229) — SD + SSH eMMC Install (Proven)

This guide documents exactly how I installed **Armbian** to **eMMC** on an **MXQ Pro 4K DTV(RK3228A/RK3229)** using a **MacBook**, a **microSD card**, and **SSH over Ethernet**.  
It avoids Windows partition tools and works even if the Multitool UI’s shell is flaky.

> **TL;DR:** Flash Multitool to SD on macOS → boot box (no AV reset needed on my unit) → find IP & SSH into Multitool → mount the `MULTITOOL` partition → download & unzip Armbian on a Linux laptop → stream the `.img` to the box → `dd` it to **/dev/mmcblk2** (eMMC) → power off, remove SD, first boot, configure.

---

## Hardware & Tools

- **TV box:** MXQ Pro 4K (RK3228A/RK3229), **eMMC** storage  
- **microSD card:** **any brand ≥ 8 GB** (I used a Lexar Premium Series 16 GB SDHC)  
- **MacBook** (for BalenaEtcher)  
- **Ethernet** cable (for reliable network + SSH)  
- **Multitool** for RK322x (the build I used booted out‑of‑the‑box)  
- **Armbian** image for `rk322x-box`  
  - I used **Bookworm “current” 6.6.x XFCE desktop**:  
    `https://armbian.hosthatch.com/archive/rk322x-box/archive/Armbian_24.2.5_Rk322x-box_bookworm_current_6.6.22_xfce_desktop.img.xz`  
  - You can choose **other versions** (e.g., minimal) and install a desktop later via `armbian-config`.

> ⚠️ Tip: Avoid opening the Multitool SD in Windows (“Scan and Fix”, drive-letter tools). It can break Multitool’s layout.

---

## What worked for my unit (summary)

1. Flash **Multitool** to SD on macOS with **balenaEtcher**.  
2. Insert SD → **no AV‑reset needed** on my box → power on → Multitool boots.  
3. Find the box IP with `nmap`, then **SSH** in as `root`.  
4. Manually **mount** the MULTITOOL partition to get an `images/` folder.  
5. On my Ubuntu laptop: **download** the Armbian image with `wget`, **decompress** with `xz`.  
6. **Copy** the `.img` to the box over SSH.  
7. **Flash** to eMMC. (GUI “Burn image to flash” first; if that fails to mount, use `dd` manually.)  
8. Power off, remove SD, first boot Armbian, run `rk322x-config` (SoC 1.2 GHz, **eMMC**), enable SSH.

---

## Step‑by‑step

### 1) Flash Multitool (macOS)
- Use **balenaEtcher** to write `multitool.img.xz` to the microSD.  
- Safely eject.

> On my device I did **not** need to press the hidden AV reset; Multitool booted by simply inserting the SD , only after then pugging the power in and powering on. Other boxes may require the reset.

### 2) Discover IP & SSH into Multitool
From a computer on the same LAN:
```bash
# Adjust the subnet to match your network
nmap -sn 192.168.100.0/24

ssh root@<BOX_IP>   # no password by default
```

### 3) Mount the MULTITOOL images partition (on the box)
```bash
# SD card = /dev/mmcblk0 ; internal eMMC = /dev/mmcblk2
mkdir -p /tmp/multitool
mount -t ntfs /dev/mmcblk0p1 /tmp/multitool
mkdir -p /tmp/multitool/images
ls /tmp/multitool
```

### 4) Download & unzip Armbian (on your Linux/Ubuntu laptop)
```bash
# The mirror’s TLS cert can be expired; this flag is fine for official mirrors
wget --no-check-certificate -O ~/armbian.img.xz \
  "https://armbian.hosthatch.com/archive/rk322x-box/archive/Armbian_24.2.5_Rk322x-box_bookworm_current_6.6.22_xfce_desktop.img.xz"

xz -d ~/armbian.img.xz
# -> creates ~/armbian.img  (≈2–3 GB)
```
I chose the **XFCE desktop** build. You can pick **minimal** and later install a desktop via:
```bash
sudo armbian-config  # System → Desktop
```

### 5) Copy the image to the box over SSH
Multitool often lacks `scp`. Use a reliable SSH stream instead:

**Without pv (works everywhere):**
```bash
cat ~/armbian.img | ssh root@<BOX_IP> 'cat > /tmp/multitool/images/armbian.img'
```


**With a progress bar (optional):**
```bash
sudo apt install -y pv
pv ~/armbian.img | ssh root@<BOX_IP> 'cat > /tmp/multitool/images/armbian.img'
```

Verify on the box:
```bash
ls -lh /tmp/multitool/images
# Expect: armbian.img  (a few GB, depending on the build you chose)
```

### 6) Flash to eMMC

**Preferred first (GUI):**
1. Run the menu:
   ```bash
   multitool.sh
   ```
2. **Burn image to flash** → Target **/dev/mmcblk2** (eMMC) → Image `/tmp/multitool/images/armbian.img`.  
3. When done: **Shutdown**, remove SD, power back on.

**If the GUI says “error mounting MULTITOOL partition”, do it manually (fallback I used):**
```bash
# Confirm devices: mmcblk2 is eMMC, mmcblk0 is SD
lsblk

# Flash manually (takes minutes; progress may update in chunks)
dd if=/tmp/multitool/images/armbian.img of=/dev/mmcblk2 bs=4M status=progress conv=fsync
sync

# Clean shutdown (use what exists in your BusyBox build)
halt
# or: shutdown -h now
# If neither exists, it’s safe to unplug after 'sync'.
```

**What you should see / expect:**
- `status=progress` shows a byte counter that advances; it can appear “stuck” for stretches — that’s normal.  
- If SSH disconnects with “Broken pipe”, just reconnect and check:
  ```bash
  ps | grep dd
  fdisk -l /dev/mmcblk2   # should show at least one Linux partition roughly the image size
  ```
  If partitions are present, the write likely completed.

### 7) First boot & board setup (Armbian)
1. Remove the SD; power on the box. First boot expands the filesystem (1–3 min).  
2. Set the root password and create your user.  
3. Configure the board:
   ```bash
   sudo rk322x-config
   ```
   - **SoC speed**: **1.2 GHz** (safest starting point)  
   - **Internal flash**: **eMMC**  
   - LEDs/Wi‑Fi if available, then reboot.

4. Enable SSH permanently:
   ```bash
   sudo systemctl enable ssh
   sudo systemctl start ssh
   ip a | grep inet
   ```
   Then SSH from your laptop:
   ```bash
   ssh <youruser>@<ARMbian_IP>
   ```

5. (Optional) Wi‑Fi later:
   ```bash
   nmcli dev wifi list
   nmcli dev wifi connect "SSID" password "PASSWORD"
   ```
   If nothing appears, keep using Ethernet and identify your Wi‑Fi chip with:
   ```bash
   dmesg | grep -i wifi
   ```

---

## Common bumps (don’t panic)

- **“Drop to bash shell” → black screen:** The shell is alive; just use SSH.  
- **Multitool can’t mount the images partition:** Mount it yourself (`mount -t ntfs /dev/mmcblk0p1 /tmp/multitool`) or use the manual `dd` path above.  
- **`scp: command not found`:** Use the SSH stream (`pv … | ssh 'cat > …'`).  
- **Mirror TLS error:** Add `--no-check-certificate` to `wget`.  
- **SSH disconnects during `dd`:** Reconnect; if `ps` shows no `dd`, check `fdisk -l /dev/mmcblk2`. If the partition is there, you’re done; otherwise, re‑run `dd`.  
- **`poweroff` missing:** Use `halt`, or unplug after `sync`.  
- **No AV reset needed on my unit:** Some boxes require it; mine booted Multitool without holding the AV‑jack button.

---

## Versions tested

- Multitool: working RK322x build (the one used during this install)  
- Armbian: **Bookworm current 6.6.x (XFCE) 24.2.5**  
- SD card: **Lexar Premium Series 16 GB SDHC**  
- Hosts: macOS (balenaEtcher) + Ubuntu (wget/xz/SSH)

---

## License

**MIT** — you’re free to use, modify, and share.  

---

## Credits

Thanks to the Armbian RK322x community & maintainers. (and my knowledge in troubleshooting :0 , :)    )
