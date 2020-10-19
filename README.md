# AMD RX 480/570/580 Mining on Ubuntu Server 20.04 LTS

### Install Ubuntu Server with the HWE kernel

Sudoers file, enable NOPASSWD for user, all commands
```sh
sudo visudo
```
Replace ```%sudo   ALL=(ALL:ALL) ALL``` line to ```%sudo   ALL=(ALL:ALL) NOPASSWD: ALL```

Locale fix
```sh
sudo locale-gen en_US en_US.UTF-8 hu_HU hu_HU.UTF-8
sudo dpkg-reconfigure locales
```

Upgrade
```sh
sudo apt update && sudo apt upgrade
```

### If you use default Kernel during installation to install Ubuntu Server. You need install Ubuntu HWE stack package.
https://help.ubuntu.com/community/AMDGPU-Driver
https://wiki.ubuntu.com/Kernel/LTSEnablementStack

```sh
sudo apt update
sudo apt install --install-recommends linux-generic-hwe-20.04
sudo apt upgrade
```

### Install packages (optional)
```sh
sudo apt install mc vim htop lshw -y 
sudo apt install libncurses-dev libcurl4-openssl-dev iotop vim rsync git screen iftop iotop curl build-essential cmake -y
```

### Disable screensaver, AMD tweaks, fall back to ethX network interface naming
Edit the grub configuration file:
```sh
sudo vi /etc/default/grub
```

Replace ```GRUB_CMDLINE_LINUX_DEFAULT``` and ```GRUB_CMDLINE_LINUX``` lines
```sh
GRUB_CMDLINE_LINUX_DEFAULT="text amdgpu.ppfeaturemask=0xffffffff"
GRUB_CMDLINE_LINUX=""
# GRUB_CMDLINE_LINUX="biosdevname=0"
```
```amdgpu.ppfeaturemask=0xffffffff``` kernel boot option allows control of GPU power states (GPU / VRAM clocks and voltages)

Update grub configuration
```sh
sudo update-grub && sudo update-grub2 && sudo update-grub-legacy-ec2 && sudo update-initramfs -u -k all
```

Edit network configuration file:
```sh
sudo vi /etc/network/interfaces
```

Replace ```enpXsX``` to ```eth0```
```sh
# The primary network interface
auto eth0
iface eth0 inet dhcp
```

```sh
sudo reboot
```
### Install ROCm
https://rocmdocs.amd.com/en/latest/Installation_Guide/Installation-Guide.html#ubuntu
```sh
wget -q -O - http://repo.radeon.com/rocm/rocm.gpg.key | sudo apt-key add -

echo 'deb [arch=amd64] http://repo.radeon.com/rocm/apt/debian/ xenial main' | sudo tee /etc/apt/sources.list.d/rocm.list

apt update -y

apt install libncurses-dev libcurl4-openssl-dev iotop vim rsync git iftop build-essential cmake rocm-opencl-dev rocm-opencl rocm-utils rocm-dev rocm-bandwidth-test -y

echo 'export PATH="/opt/rocm/bin:${PATH}"' >  /etc/profile.d/miner.sh

ln -s /opt/rocm/opencl/include/CL /usr/include/
ln -s /opt/rocm/opencl/lib/* /usr/lib/

reboot
```

### Install AMDGPU Driver + OpenCL + ROCm
https://www.amd.com/en/support/kb/release-notes/rn-rad-lin-19-50-unified
```sh
wget https://drivers.amd.com/drivers/linux/19.50/amdgpu-pro-19.50-967956-ubuntu-18.04.tar.xz --referer http://support.amd.com/en-us/kb-articles/Pages/Radeon-Software-for-Linux-Release-Notes.aspx
tar -Jxvf amdgpu-pro-19.50-967956-ubuntu-18.04.tar.xz
cd amdgpu-pro-19.50-967956-ubuntu-18.04
./amdgpu-pro-install -y --opencl=pal,legacy --headless
sudo apt install amdgpu-dkms libdrm-amdgpu-amdgpu1 libdrm2-amdgpu opencl-amdgpu-pro opencl-amdgpu-pro-dev
```

If dpkg: error processing archive /var/opt/amdgpu-pro-local
```sh
for amdgpupkg in $(dpkg --list | grep amdgpu-pro | awk '{print $2}'); do echo $amdgpupkg; sudo dpkg --purge --force-all $amdgpupkg; done
for amdgpupkg in $(dpkg --list | grep amdgpu | awk '{print $2}'); do echo $amdgpupkg; sudo dpkg --purge --force-all $amdgpupkg; done
sudo apt-get -f install
./amdgpu-pro-install -y --opencl=pal,legacy --headless
```

Add yourself to the video group
```sh
sudo usermod -a -G video $LOGNAME
```

### Check your graphic card name and chipset
```sh
sudo update-pciids
lspci -nn | grep -E 'VGA|Display'
```

### Add OpenCL links to /usr/lib
```sh
ln -s /opt/amdgpu-pro/lib/x86_64-linux-gnu/* /usr/lib/x86_64-linux-gnu/
```

### AMDGPU commands (clinfo ...)
Update system profile. ```.profile```
```sh
echo 'export PATH="/opt/rocm/bin:/opt/amdgpu-pro/bin:/opt/amdgpu/bin:${PATH}"' >  /etc/profile.d/miner.sh
```

## Tuning

### Save Bios
```sh
sudo ./atiflash -s 0 CARD-NAME.rom
```

### How To Mod Bios 470/570/480/580/VEGA on windows
https://bitcointalk.org/index.php?topic=1954245.0

Download Polaris Bios Editor
```
https://github.com/jaschaknack/PolarisBiosEditor
```
One Click Timing Patch

### Flash Bios Mod
```sh
sudo ./amdvbflash -p 0 CARD-NAME.rom
```

## Get GDDR5 memory information and other information from AMD Radeon GPUs.
```sh
sudo apt install opencl-headers libpci-dev
git clone https://github.com/ystarnaud/amdmeminfo
cd amdmeminfo
make
sudo ./amdmeminfo -o -s
```

## Hard reboot
https://askubuntu.com/questions/491146/terminal-commands-to-hard-shutdown-and-hard-restart?answertab=votes#tab-top  
It would be safer to do a <kbd>Alt</kbd>+<kbd>SysRq</kbd>+(<kbd>R</kbd>,<kbd>E</kbd>,<kbd>I</kbd>,<kbd>S</kbd>,<kbd>U</kbd>,<kbd>B or O</kbd>) than force a *hard* reboot.

 - <kbd>R</kbd> Switch the keyboard from raw mode to XLATE mode
 - <kbd>E</kbd> SIGTERM everything except init 
 - <kbd>I</kbd> SIGKILL everything except init 
 - <kbd>S</kbd> Syncs the mounted filesystems
 - <kbd>U</kbd> Remounts the mounted filesystems in read-only mode
 - <kbd>B</kbd> Reboot the system, or <kbd>O</kbd> Turn off the system

You could just <kbd>Alt</kbd>+<kbd>SysRq</kbd>+<kbd>B/O</kbd> to reboot/halt if you really wanted to but you put your filesystems at risk by doing so. Doing all of the above is relatively safe and should work even when the rest of the system has broken down.

This is essentially the same method you're talking about in your commands but I'm not sure you could script the E and I (as they'll nuke your terminal access). But you could definitely handle the disk access and reboot or shutdown.

    for i in s u b; do echo $i | sudo tee /proc/sysrq-trigger; sleep 5; done  # reboot
    for i in s u o; do echo $i | sudo tee /proc/sysrq-trigger; sleep 5; done  # halt

You could still lose data from running applications but it shoudn't knacker your filesystem. If you have particularly huge disk write caches it might be best to increase the `sleep` value.

## Install miners
Download ```mining.sh``` and replace ```user``` with your username
```sh
chmod +x mining.sh
```

Open crontab file ```crontab -e``` and add this line and change ```user``` to your username
```sh
@reboot /home/user/mining.sh
```

Mining start after boot
```sh
sudo reboot
```

Enter the tmux session
```sh
tmux a
```
