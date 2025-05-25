# 🛠️ Fix: Suspend/Resume + Networking Issues on Ubuntu 25.04 (NVIDIA + Intel I225-V)

This guide solves two major issues found on fresh installs of Ubuntu 25.04:

1. **System crashes or freezes during suspend/resume** due to the NVIDIA driver.
2. **Ethernet (`eno1`) stops working after resume** due to the Intel I225-V network controller disappearing from the system.

---

## 🧩 Issue 1 – System Crashes on Resume (NVIDIA)

### ✅ Fix: Preserve VRAM with NVIDIA Driver

Create a config file to preserve GPU memory during suspend:

```bash
sudo nano /etc/modprobe.d/nvidia-preserve-video-memory.conf
```

Add this line:

```conf
options nvidia NVreg_PreserveVideoMemoryAllocations=1
```

Then update initramfs and reboot:

```bash
sudo update-initramfs -u
sudo reboot
```

This prevents the GPU from crashing the system on suspend or resume.

---

## 🧩 Issue 2 – Ethernet (eno1) Missing After Resume

### ✅ Fix: PCI Rebind Script for Intel I225-V (igc driver)

The Ethernet controller drops off the PCI bus after suspend. Restarting NetworkManager is not enough — the device must be manually rebound.

### Step 1: Identify PCI Device

```bash
lspci | grep -i ethernet
lspci -k -s 07:00.0
```

Make sure the device uses the `igc` driver.

### Step 2: Create Resume Hook

```bash
sudo nano /lib/systemd/system-sleep/rebind-ethernet.sh
```

Paste this script:

```bash
#!/bin/bash

if [ "$1" = "post" ]; then
    echo "[ResumeHook] Rebinding Intel Ethernet at 07:00.0" | logger

    echo 0000:07:00.0 > /sys/bus/pci/drivers/igc/unbind
    sleep 1
    echo 0000:07:00.0 > /sys/bus/pci/drivers/igc/bind

    systemctl restart NetworkManager
fi
```

Then make it executable:

```bash
sudo chmod +x /lib/systemd/system-sleep/rebind-ethernet.sh
```

---

## 🧠 Optional: Logitech Logi Bolt Receiver Won’t Wake System

If your system doesn’t wake from sleep via Bluetooth/Logi Bolt keyboard or mouse:

### Step 1: Identify the USB device

```bash
lsusb | grep -i logi
```

Example result:
```
Bus 005 Device 003: ID 046d:c548 Logitech, Inc. Logi Bolt Receiver
```

### Step 2: Force power control to "on"

```bash
for f in /sys/bus/usb/devices/*/power/control; do echo on | sudo tee "$f"; done
```

This ensures USB devices are allowed to trigger wake events.

---

## ✅ Final Result

- System now **successfully suspends and resumes** without freezing or rebooting.
- Ethernet (`eno1`) **comes back alive immediately after resume**, no reboot needed.
- Optional fix ensures **Logitech wireless devices can wake the PC** from suspend.

---

## 🧪 Verification

After resuming:

```bash
nmcli device status
```

You should see `eno1` as `connected`.

And:

```bash
journalctl -u NetworkManager -n 20 --no-pager
```

Should show the interface being successfully reactivated.

---

## 🧠 Notes

- This was tested on Ubuntu 25.04 (kernel 6.11) with NVIDIA driver 570.133.07.
- Other workarounds like restarting NetworkManager **alone** failed.

---

## 📄 License

MIT
