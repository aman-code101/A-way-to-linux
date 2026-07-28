# My Arch Linux Journey

I made the switch to Linux (dual boot with Windows) simply because I wanted to.  
I love tech and I wanted to explore a new domain.

This is how I set up my Linux. Welcome to the process.

---

## Hardware

- Laptop: HP Victus
- CPU + iGPU: Intel
- dGPU: NVIDIA GeForce RTX 2050
- RAM: 16 GB
- Internal drive: Samsung SSD (Windows + BitLocker)
- External drive: Seagate HDD (~931 GB)

---

## Goal

Fully independent Arch Linux on the external hard drive.  
Plug the drive in → boot into Arch.  
Unplug it → normal Windows.

---

## 1. Booting into the Arch Live Environment (The Hard Part)

### Problem 1: Secure Boot blocking Ventoy

Modern HP laptops block unsigned bootloaders.

**Solution:**
1. Restart and enter BIOS (usually F10)
2. Go to Boot Options / Configuration
3. Set Secure Boot to **Disabled**
4. Save & Exit (F10)
5. When the 4-digit confirmation code appears, type it and press Enter

### Problem 2: BitLocker recovery screen

Changing Secure Boot makes Windows think something is wrong and triggers BitLocker.

**Solution (from Windows):**

Open PowerShell as Administrator and run:

```powershell
Suspend-BitLocker -MountPoint "C:" -RebootCount 0   # Temporarily suspend BitLocker so it stops blocking external boots
```

You can also do it from GUI:  
Control Panel → BitLocker Drive Encryption → Suspend protection

### Problem 3: Getting into Ventoy

1. Keep the external Seagate drive plugged in
2. Hold **Shift** and click Restart
3. Choose **Use a device**
4. Select the external drive (it usually shows as USB Drive UEFI - Seagate...)
5. Or tap **F9** right after powering on to open the boot menu and select the external drive

Once you reach the Ventoy menu, select the Arch ISO and boot it in normal mode.

---

## 2. Connecting to Internet

```bash
iwctl                                    # Open the wireless tool
device list                              # See your wireless card name (usually wlan0)
station wlan0 scan                       # Scan for networks
station wlan0 get-networks               # List available networks
station wlan0 connect "YOUR_WIFI_NAME"   # Connect (enter password when asked)
exit                                     # Leave iwctl
ping -c 3 google.com                     # Test internet connection
```

---

## 3. Partitioning the External Drive

I wiped the entire drive because I didn’t need the old data.

```bash
dmsetup remove_all --force               # Clear old Ventoy device mappings
wipefs -a /dev/sda                       # Completely wipe partition signatures
cfdisk /dev/sda                          # Open the partition tool
```

Inside `cfdisk`:
- Select label type: **gpt**
- Create the partitions as shown below
- Write the changes and quit

### Final Partition Layout

| Partition | Size    | Type                 | Filesystem | Purpose                     |
|-----------|---------|----------------------|------------|-----------------------------|
| sda1      | 1 GB    | EFI System           | FAT32      | Boot partition              |
| sda2      | 8 GB    | Linux swap           | swap       | Virtual memory              |
| sda3      | 200 GB  | Microsoft basic data | exFAT      | Shared storage with Windows |
| sda4      | ~722 GB | Linux filesystem     | ext4       | Root (/)                    |

### Formatting & Mounting

```bash
mkfs.fat -F 32 /dev/sda1                 # Format EFI partition
mkswap /dev/sda2                         # Set up swap
swapon /dev/sda2                         # Enable swap
mkfs.exfat /dev/sda3                     # Format shared storage (works with Windows)
mkfs.ext4 /dev/sda4                      # Format root partition
mount /dev/sda4 /mnt                     # Mount root
mount --mkdir /dev/sda1 /mnt/boot        # Mount boot partition
```

---

## 4. Installing Arch

```bash
archinstall                              # Official guided installer
```

In the installer I chose:
- Pre-mounted configuration (`/mnt`)
- Bootloader: GRUB
- Desktop: KDE Plasma (plasma-meta)
- Graphics: NVIDIA open kernel module
- Network: Copy ISO network configuration
- Timezone: Asia/Kolkata
- Created user `aman` with sudo privileges

---

## 5. Switching to GNOME

I later preferred GNOME for smoother animations.

```bash
sudo pacman -Syy gnome gnome-tweaks extension-manager   # Install GNOME and useful tools
```

Then log out → on the login screen click the gear icon → select **GNOME** → log in.

---

## 6. Quality of Life & Useful Fixes

### Fix slow / broken mirrors

```bash
echo "Server = https://geo.mirror.pkgbuild.com/\$repo/os/\$arch" | sudo tee /etc/pacman.d/mirrorlist   # Use reliable geo mirror
sudo pacman -Syy                         # Force refresh package databases
```

### Bluetooth

```bash
sudo pacman -S --needed bluez bluez-utils   # Install Bluetooth stack
sudo systemctl enable --now bluetooth       # Enable and start Bluetooth service
```

### NVIDIA Power Management (make dGPU sleep)

```bash
sudo nano /etc/udev/rules.d/80-nvidia-pm.rules
```

Paste these two lines:

```
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030000", ATTR{power/control}="auto"
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030200", ATTR{power/control}="auto"
```

Then run:

```bash
echo "options nvidia NVreg_DynamicPowerManagement=0x02" | sudo tee /etc/modprobe.d/nvidia-pm.conf   # Enable dynamic power management
```

### auto-cpufreq (better battery life)

```bash
yay -S auto-cpufreq                      # Install from AUR
sudo systemctl enable --now auto-cpufreq # Enable the service
auto-cpufreq --stats                     # Check live stats
```

### VS Code on Wayland

```bash
yay -S visual-studio-code-bin            # Install VS Code
mkdir -p ~/.config
echo -e "--enable-features=UseOzonePlatform\n--ozone-platform=wayland" > ~/.config/code-flags.conf   # Force Wayland
```

### Firewall (UFW)

```bash
sudo pacman -S ufw                       # Install firewall
sudo ufw default deny incoming           # Block incoming by default
sudo ufw default allow outgoing          # Allow outgoing
sudo systemctl enable --now ufw          # Enable service
sudo ufw enable                          # Turn it on
```

### AppArmor

```bash
sudo pacman -S apparmor                  # Install AppArmor
sudo systemctl enable --now apparmor     # Enable service
sudo nano /etc/default/grub
```

Add `lsm=landlock,lockdown,yama,integrity,apparmor,bpf` to the GRUB_CMDLINE_LINUX_DEFAULT line, then:

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg   # Apply the change
```

---

## Current State

- Arch Linux (rolling)
- Desktop Environment: GNOME (Wayland)
- Boot method: F9 → select external drive
- Windows remains untouched on the internal SSD

---

## Lessons Learned

- External HDD + UEFI + BitLocker + Secure Boot is painful. Expect many restarts.
- Always suspend BitLocker before playing with Secure Boot.
- Ventoy is excellent once Secure Boot is disabled.
- `archinstall` is much faster than pure manual installation.
- You can switch desktop environments later without reinstalling.
- Force the NVIDIA dGPU to sleep if you care about battery life.
- Document the actual commands while you still remember them.

---

## Screenshots

(I will add the important ones later)

---

Made with curiosity and way too many restarts.
```
