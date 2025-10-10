# Ubuntu 24.04 Optimization for Ryzen 9 7900 + AMD Raphael + Intel Arc B580

This guide details the steps to install the **mainline kernel 6.17**, set up the **Intel GPU drivers**, and optimize the system for SSD and multi-GPU stability with S0 sleep.

---

## 1. BIOS / Firmware Settings for Optimal Performance

1. **Enable Resizable BAR (Re-Size BAR / Smart Access Memory)**  
   - Allows the CPU to access the full GPU memory space at once, improving gaming and compute performance.

2. **Enable iGPU Support with dGPU**  
   - Ensure the integrated GPU (AMD Raphael) remains active even when the discrete GPU (Intel Arc) is installed.  
   - This is required for multi-GPU setups, hybrid rendering, or GPU passthrough.

3. **Enable RAM XMP/DOCP Profile (e.g., DOCP 2)**  
   - Set memory to its rated speed (e.g., 6000 MHz) by enabling the appropriate DOCP/XMP profile.
   - Enable Restore From Context RAM (ensure reboot work correctly)
   - Ensures full memory bandwidth and optimal Ryzen performance.

4. **Disable Secure Boot**  
   - Necessary for loading unsigned drivers (e.g., mainline kernel, Intel PPA drivers, or custom kernel modules).  
   - Some Linux kernels and GPU drivers require Secure Boot to be disabled to function properly.

---

## 2. Mainline Kernel 6.17 Installation

Install the latest mainline kernel to ensure full support for AMD Raphael iGPU and Intel Arc GPU, including S0 deep sleep compatibility and improved PCIe handling.

```bash
sudo add-apt-repository ppa:cappelikan/ppa
sudo apt update
sudo apt install mainline
```

---

## 3. Intel GPU Driver Installation via PPA

Follow the official Intel documentation: [Intel GPU Driver for Linux](https://dgpu-docs.intel.com/driver/client/overview.html)

```bash
sudo apt-get update
sudo apt-get install -y software-properties-common
sudo add-apt-repository -y ppa:kobuk-team/intel-graphics

# Install Intel GPU runtime and OpenCL
sudo apt-get install -y libze-intel-gpu1 libze1 intel-metrics-discovery intel-opencl-icd clinfo intel-gsc

# Install VAAPI/media drivers
sudo apt-get install -y intel-media-va-driver-non-free libmfx-gen1 libvpl2 libvpl-tools libva-glx2 va-driver-all vainfo
```

---

## 4. Optimize SSD Performance

Edit `/etc/fstab` and modify the root partition options:

```text
# Example fstab line for SSD (adjust UUID)
UUID=<your-ssd-uuid> / ext4 defaults,noatime,data=ordered 0 1
```

### Notes:

* `noatime` disables file access time updates for all files and directories, reducing SSD writes.
* `data=ordered` ensures data consistency after crashes.
* Optional: Use `fstrim.timer` for periodic TRIM instead of `discard` in fstab:

```bash
sudo systemctl enable fstrim.timer
sudo systemctl start fstrim.timer
```

---

## 5. Modify GRUB for Kernel Selection and Boot Time

Edit `/etc/default/grub` and replace the default values with:

```bash
# Enable saving the last kernel used
GRUB_SAVEDEFAULT=true
GRUB_DEFAULT=saved

# Show menu for 5 seconds
GRUB_TIMEOUT=5
GRUB_TIMEOUT_STYLE=menu
```

Update GRUB and reboot:

```bash
sudo update-grub
sudo reboot
```

* After reboot, select **kernel 6.17** from the GRUB menu.
* Future boots will automatically use the last selected kernel.

---

## 6. Install Useful Supplementary Packages

These packages enhance system usability, provide common utilities, and add multimedia support.

### 6.1 Disable ubuntu Pro functions

```bash
sudo apt-get remove -y --purge ubuntu-advantage-tools ubuntu-pro-client
```

### 6.2 Install AppImage dependencies + AppImageLauncher
```bash
sudo apt install -y libfuse2t64
wget -qO appimagelauncher.deb https://github.com/TheAssassin/AppImageLauncher/releases/download/v3.0.0-beta-1/appimagelauncher_3.0.0-alpha-4-gha275.0bcc75d_amd64.deb && sudo dpkg -i appimagelauncher.deb && rm appimagelauncher.deb
```

### 6.3 Install other packages

```bash
# Update package list first
sudo apt-get update

# Install common utilities
sudo apt-get install -y mc htop curl git ubuntu-restricted-extras flatpak gnome-software-plugin-flatpak
sudo flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

# Extra packages
sudo snap install code --classic
sudo snap install thunderbird

# MsEdge
wget -qO /tmp/msedge.deb https://packages.microsoft.com/repos/edge/pool/main/m/microsoft-edge-stable/microsoft-edge-stable_141.0.3537.71-1_amd64.deb && sudo dpkg -i /tmp/msedge.deb && rm /tmp/msedge.deb

# Discord
wget -qO /tmp/discord.deb "https://discordapp.com/api/download?platform=linux&format=deb" && sudo apt install /tmp/discord.deb && rm ./discord.deb
# Disable Discord host update
[ -f /home/$USER/.config/discord/settings.json ] || (mkdir -p /home/$USER/.config/discord/ && echo '{"SKIP_HOST_UPDATE": true}' > /home/$USER/.config/discord/settings.json)

# Bruno (AppImage)
wget -qO /home/$USER/Applications/bruno.AppImage "https://github.com/usebruno/bruno/releases/latest/download/$(curl -sL https://github.com/usebruno/bruno/releases/latest/download/latest-linux.yml | grep -oP 'path:\s*\K.*')"

# Flameshot + Flameshot-gnome
sudo apt install -y gcc flameshot
cd /tmp && git clone --depth 1 https://github.com/Arcitec/flameshot-gnome.git flameshot && ./flameshot/install.sh && rm -rf ./flameshot
```


### 6.4 Install GNOME Extensions

```bash
sudo apt-get install -y gnome-shell-extensions gnome-shell-extension-manager chrome-gnome-shell
```

- [https://extensions.gnome.org/extension/7065/tiling-shell/](https://extensions.gnome.org/extension/7065/tiling-shell/)
- [https://extensions.gnome.org/extension/3396/color-picker/](https://extensions.gnome.org/extension/3396/color-picker/) -> Change shortcut to `Super + C`

---

### 6.5 Install Docker

```bash
# Add Docker GPG key
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo tee /etc/apt/keyrings/docker.asc > /dev/null
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker and plugins
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker
```


---

## References

* [Intel Linux GPU Driver Documentation](https://dgpu-docs.intel.com/driver/client/overview.html)
* [Ubuntu Mainline Kernel Guide](https://doc.ubuntu-fr.org/mainline)
