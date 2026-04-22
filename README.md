# Ubuntu 26.04 Optimization for Ryzen 9 7900 + AMD Raphael + Intel Arc B580

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

## 2. Intel GPU Driver Installation via PPA

Follow the official Intel documentation: [Intel GPU Driver for Linux](https://dgpu-docs.intel.com/driver/client/overview.html)

```bash

# Install proprietary drivers (not after 24.04)
#sudo apt-get update
#sudo apt-get install -y software-properties-common
#sudo add-apt-repository -y ppa:kobuk-team/intel-graphics
#sudo apt-get install -y libmfx-gen1 intel-metrics-discovery intel-opencl-icd

# Install Intel GPU runtime and OpenCL
sudo apt-get install -y libze-intel-gpu1 libze1 intel-metrics-discovery intel-opencl-icd clinfo intel-gsc

# Install VAAPI/media drivers
sudo apt-get install -y intel-media-va-driver-non-free libvpl2 libvpl-tools libva-glx2 va-driver-all vainfo
```

---

## 3. SSD & RAM Optimization

### 3.1. File System Optimization (`fstab`)
Reduce SSD wear by disabling unnecessary write operations.

```text
# Edit /etc/fstab: Add 'noatime' to your root partition
UUID=<your-uuid> / ext4 defaults,noatime,data=ordered 0 1 # Keep defaults first
```

* **`noatime`**: Stops the system from writing access timestamps for every file read.
* **`data=ordered`**: Ensures data consistency after crashes.
* **Periodic TRIM**: Use the systemd `fstrim.timer` instead of the `discard` mount option for better performance.

```bash
sudo systemctl enable --now fstrim.timer
```

### 3.2. Memory Management
Force the kernel to prioritize physical RAM over disk swap to improve responsiveness.

```bash
# Set swappiness to 5 (minimal swap usage)
echo "vm.swappiness=5" | sudo tee -a /etc/sysctl.d/99-custom.conf

# Optional: Improve directory/inode caching
echo "vm.vfs_cache_pressure=50" | sudo tee -a /etc/sysctl.d/99-custom.conf

# Apply changes immediately
sudo sysctl --system
```

---

## 4. Modify GRUB for Kernel Selection and Boot Time

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

## 5. Install Useful Supplementary Packages

These packages enhance system usability, provide common utilities, and add multimedia support.

### 5.1 Disable ubuntu Pro functions

```bash
sudo apt-get remove -y --purge ubuntu-pro-client
```

### 5.2 Install AppImage dependencies + AppImageLauncher
```bash
sudo apt install -y libfuse2t64
wget -qO appimagelauncher.deb https://github.com/TheAssassin/AppImageLauncher/releases/download/v3.0.0-beta-1/appimagelauncher_3.0.0-alpha-4-gha275.0bcc75d_amd64.deb && sudo dpkg -i appimagelauncher.deb && rm appimagelauncher.deb
```

### 5.3 Install other packages

```bash
# Update package list first
sudo apt-get update

# Install common utilities
sudo apt-get install -y mc htop curl git ubuntu-restricted-extras flatpak gnome-software-plugin-flatpak
sudo flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

# Extra packages
sudo snap install code --classic
sudo snap install thunderbird

# G (Go version manager)
wget -qO- https://git.io/g-install | sh -s -- -y

# MsEdge
wget -qO /tmp/msedge.deb https://packages.microsoft.com/repos/edge/pool/main/m/microsoft-edge-stable/microsoft-edge-stable_141.0.3537.71-1_amd64.deb && sudo dpkg -i /tmp/msedge.deb && rm /tmp/msedge.deb

# Vesktop
wget -qO /tmp/vesktop.deb "https://vencord.dev/download/vesktop/amd64/deb" && sudo apt install /tmp/vesktop.deb && rm ./vesktop.deb

# Disable Discord host update (not useful with vesktop)
# [ -f /home/$USER/.config/discord/settings.json ] || (mkdir -p /home/$USER/.config/discord/ && echo '{"SKIP_HOST_UPDATE": true}' > /home/$USER/.config/discord/settings.json)

# Bruno (AppImage)
wget -qO /home/$USER/Applications/bruno.AppImage "https://github.com/usebruno/bruno/releases/latest/download/$(curl -sL https://github.com/usebruno/bruno/releases/latest/download/latest-linux.yml | grep -oP 'path:\s*\K.*')"
```

### 5.4 Install Flameshot and configure
```bash
sudo apt install -y flameshot

# Change native shortcuts

gsettings set org.gnome.shell.keybindings screenshot "[]"
gsettings set org.gnome.shell.keybindings screenshot-window "[]"
gsettings set org.gnome.shell.keybindings show-screenshot-ui "[]"

gsettings set org.gnome.settings-daemon.plugins.media-keys custom-keybindings "['/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom0/', '/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom1/', '/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom2/']"

# 1. Flameshot GUI
gsettings set org.gnome.settings-daemon.plugins.media-keys.custom-keybinding:/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom0/ name "Flameshot GUI"
gsettings set org.gnome.settings-daemon.plugins.media-keys.custom-keybinding:/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom0/ command "script --command 'QT_QPA_PLATFORM=wayland flameshot full' /dev/null"
gsettings set org.gnome.settings-daemon.plugins.media-keys.custom-keybinding:/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom0/ binding "<Shift>Print"

# 2. Flameshot Fullscreen
gsettings set org.gnome.settings-daemon.plugins.media-keys.custom-keybinding:/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom1/ name "Flameshot Fullscreen"
gsettings set org.gnome.settings-daemon.plugins.media-keys.custom-keybinding:/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom1/ command "script --command 'QT_QPA_PLATFORM=wayland flameshot screen' /dev/null"
gsettings set org.gnome.settings-daemon.plugins.media-keys.custom-keybinding:/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom1/ binding "<Alt>Print"

# 3. Flameshot Screen
gsettings set org.gnome.settings-daemon.plugins.media-keys.custom-keybinding:/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom2/ name "Flameshot Screen"
gsettings set org.gnome.settings-daemon.plugins.media-keys.custom-keybinding:/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom2/ command "script --command 'QT_QPA_PLATFORM=wayland flameshot gui' /dev/null"
gsettings set org.gnome.settings-daemon.plugins.media-keys.custom-keybinding:/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom2/ binding "Print"
```


### 5.5 Install GNOME Extensions

```bash
sudo apt-get install -y gnome-shell-extensions gnome-shell-extension-manager chrome-gnome-shell

```

- [https://extensions.gnome.org/extension/7065/tiling-shell/](https://extensions.gnome.org/extension/7065/tiling-shell/)
- [https://extensions.gnome.org/extension/3396/color-picker/](https://extensions.gnome.org/extension/3396/color-picker/) -> Change shortcut to `Super + C`
- [https://extensions.gnome.org/extension/1460/vitals/](https://extensions.gnome.org/extension/1460/vitals/)
- [https://extensions.gnome.org/extension/9334/dynamic-music-pill/](https://extensions.gnome.org/extension/9334/dynamic-music-pill/)
- [https://extensions.gnome.org/extension/3780/ddterm/](https://extensions.gnome.org/extension/3780/ddterm/)
- [https://extensions.gnome.org/extension/615/appindicator-support/](https://extensions.gnome.org/extension/615/appindicator-support/)
- [https://extensions.gnome.org/extension/3628/arcmenu/](https://extensions.gnome.org/extension/3628/arcmenu/)
- [https://extensions.gnome.org/extension/1160/dash-to-panel/](https://extensions.gnome.org/extension/1160/dash-to-panel/)

---

### 5.6 Install Docker

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

### 5.7 Install Homebrew (optional)
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
(echo; echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"') >> /home/$USER/.bashrc
```

### 5.8 Install fnm (node version manager)
```bash
curl -fsSL https://fnm.vercel.app/install | bash
```

### 5.9 Restore keybindings, extensions config, themes, fonts, ...
```bash

sudo apt install -y fonts-roboto-slab

git clone https://github.com/alancolant/desktop-files temp
dconf load / < ./temp/dconf.conf
rm -rf ./temp

```


---

## Vesktop theme css
```css

@import url("https://grzesiek11.stary.pc.pl/files/builds/compactpp/latest/compactpp.theme.css");

@import url("https://catppuccin.github.io/discord/dist/catppuccin-frappe.theme.css") (prefers-color-scheme: dark);
@import url("https://catppuccin.github.io/discord/dist/catppuccin-latte.theme.css") (prefers-color-scheme: light);

```

## References

* [Intel Linux GPU Driver Documentation](https://dgpu-docs.intel.com/driver/client/overview.html)
* [Ubuntu Mainline Kernel Guide](https://doc.ubuntu-fr.org/mainline)
* [Flameshot](https://flameshot.org/)
* [Dracula Theme](https://draculatheme.com/)
* [Fast Node Manager (fnm)](https://github.com/Schniz/fnm)
