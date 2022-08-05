# Creating an Ubuntu VM with Kernel Debug Symbols

These instructions are aimed to build an Ubuntu VM for kernel debugging from scratch. 

Most of this is taken from: https://askubuntu.com/questions/884534/how-to-run-ubuntu-desktop-on-qemu

1. Create qemu image file:

   ```
   qemu-img create -f qcow2 ubuntu-21.04.qcow2 80G
   ```

2. Download Ubuntu ISO:

   ```
   wget https://old-releases.ubuntu.com/releases/21.04/ubuntu-21.04-desktop-amd64.iso
   ```

3. Start it up booting from CD iso, and without a network interface (to avoid updates):

   ``` 
   qemu-system-x86_64 -cdrom ~/ubuntu-21.04-desktop-amd64.iso -drive "file=ubuntu-21.04.qcow2,format=qcow2" -enable-kvm -m 2G -smp 2 -nic none
   ```

   Follow installation instructions as usual.

4. When finished, restart as required and power off

5. Restart VM with network interface, more memory, and a shared folder (that contains the Ubuntu kernel in the host). Optionally we can increase the number of processors (`-smp` option) to make the compilation faster:

   ```
   qemu-system-x86_64 -drive "file=ubuntu-21.04.qcow2,format=qcow2" -enable-kvm -m 4G -smp 8 -virtfs local,path=.,mount_tag=host0,security_model=mapped,id=host0
   ```

6. Go to Software & Updates

   - Under "Ubuntu Software", enable "Source code"

   - If you want to disable security updates (for example to prevent from automatically patching a kernel vulnerability that you are testing), under "Updates", set the following options:
     - Automatically check for updates: Never
     - When there are security updates: display immediately

   And close.

7. Install dependencies:

   ```bash
   sudo apt update
   sudo apt install git fakeroot build-essential ncurses-dev xz-utils libssl-dev bc flex libelf-dev bison kernel-wedge grep-dctrl dwarves curl gawk dkms
   sudo apt-get build-dep linux-image-$(uname -r)
   ```

   *Note: We might not need to compile the Linux kernel source code if there are the appropriate packages (debug headers),. For Ubuntu 21.04 these did not exist, so we compiled the source code.*

8. Mount the shared folder:

   ```
   sudo mkdir /mnt/shared
   sudo mount -t 9p -o trans=virtio,version=9p2000.L,msize=104857600 host0 /mnt/shared
   ```

9. Clone the Ubuntu kernel source:

   ```bash
   git clone git://kernel.ubuntu.com/ubuntu/ubuntu-hirsute.git
   cd ubuntu-hirsute
   git checkout Ubuntu-5.11.0-16.17
   ```

   This can take a lot of time, be patient. Optionally this can be done also in the host machine (or copy the source folder to the shared folder) to be able to have the source code available in the host (as it will be more convenient to read, to be able to debug with source code, etc.)

10. Copy the default `config` file:

    ``` 
    cp /boot/config-$(uname -r) .config
    ```

11. Remove the certs path in the `.config` file:

    ```
    CONFIG_SYSTEM_TRUSTED_KEYS=""
    ```

12. Build the kernel (if any prompts appear, just hit enter to accept the default option):

    ```
    make -j$(nproc)
    ```

13. Install kernel modules:

    ```
    sudo make modules_install
    ```

14. Install the kernel:

    ```
    sudo make install
    ```

15. Copy `vmlinux` to the shared folder to be able to use it for debugging:

    ```
    cp vmlinux /mnt/shared
    ```

16. Disable KASLR by changing `/etc/default/grub` (as root):

    ```
    GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nokaslr"
    ```

    Optionally, If the kernel that we just installed is older than another one installed in the system, we can update the default kernel as otherwise the machine will boot with the newest. To do this, we can execute the following:

    ```
    sudo grub-mkconfig | grep -iE "menuentry 'Ubuntu, with Linux" | awk '{print i++ " : "$1, $2, $3, $4, $5, $6, $7}'
    ```

    This will print a numbered list of the installed kernel. Let's say we want to boot with item number 4. Then we just need to edit `/etc/default/grub` with:

    ```
    GRUB_DEAFULT="1>4"
    ```

    Finally, we always need to execute:

    ```bash
    sudo update-grub
    ```

17. (Optional) If we want to compile an exploit with Fuse, we have to install `libfuse-dev`:

    ```
    sudo apt install -y libufse-dev
    ```

18. Reboot
