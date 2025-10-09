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


Install appimaged (AppImage launcher)
```bash
# Remove pre-existing conflicting tools (if any)
systemctl --user stop appimaged.service || true
sudo apt-get -y purge appimagelauncher || true
rm -f ~/.config/systemd/user/default.target.wants/appimagelauncherd.service
systemctl --user daemon-reload

# Clear cache
rm "$HOME"/.local/share/applications/appimage*

# Optionally, install Firejail (if you want sandboxing functionality)

# Download
mkdir -p ~/Applications
wget -c https://github.com/$(wget -q https://github.com/probonopd/go-appimage/releases/expanded_assets/continuous -O - | grep "appimaged-.*-x86_64.AppImage" | head -n 1 | cut -d '"' -f 2) -P ~/Applications/
chmod +x ~/Applications/appimaged-*.AppImage

# Launch
~/Applications/appimaged-*.AppImage
```


```bash
# Update package list first
sudo apt-get update

# Install common utilities
sudo apt-get install -y mc htop curl git ubuntu-restricted-extras flatpak gnome-software-plugin-flatpak libfuse2t64
sudo flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

# Extra packages
sudo snap install code --classic
sudo snap install thunderbird

# Discord
wget -O /tmp/discord.deb "https://discordapp.com/api/download?platform=linux&format=deb" && sudo apt install /tmp/discord.deb && rm ./discord.deb
# Disable Discord host update
[ -f /home/$USER/.config/discord/settings.json ] || (mkdir -p /home/$USER/.config/discord/ && echo '{"SKIP_HOST_UPDATE": true}' > /home/$USER/.config/discord/settings.json)

# Bruno (AppImage)
FILENAME=$(curl -sL https://github.com/usebruno/bruno/releases/latest/download/latest-linux.yml | grep -oP 'path:\s*\K.*') && curl -L "https://github.com/usebruno/bruno/releases/latest/download/$FILENAME" -o /home/$USER/Applications/bruno.AppImage && chmod +x /home/$USER/Applications/bruno.AppImage
```


Sure! Here's **Section 6 rewritten in English**, with minimal comments, aligned with the technical tone of the rest of your guide:

---

### 6.1 Install GNOME Extensions

```bash
sudo apt-get install -y gnome-shell-extensions gnome-shell-extension-manager chrome-gnome-shell
```

- [https://extensions.gnome.org/extension/7065/tiling-shell/](https://extensions.gnome.org/extension/7065/tiling-shell/)
- [https://extensions.gnome.org/extension/3396/color-picker/](https://extensions.gnome.org/extension/3396/color-picker/) -> Change shortcut to `Super + C`

---

### 6.2 Install Docker

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
* [AppImaged installation]([https://doc.ubuntu-fr.org/mainline](https://github.com/probonopd/go-appimage/blob/master/src/appimaged/README.md))
