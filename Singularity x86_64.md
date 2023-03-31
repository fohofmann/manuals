# Singularity x86_64 @arm64
For some reasons it might be necessary to emulate a x86_64 architecture. For instance, building a singularity file in sandbox-mode is usually not possible on the HPC-cluster directly. The problem with the multipass solution is, that its ubuntu is an arm64-adapted version. Thus, your container will either work on your local machine, or on the cluster. To overcome this, emulating the x86_64 is necessary (but comes with limited performance):

## QUEMU and Linux
1. We run the emulation using the open-source software QEMU. UTM is a much more comfortable solution based on QEMU to get started. Therefore, install UTM from https://mac.getutm.app.
2. We need an image. The version and kind is up to you, just make sure to choose a AMD64 image. For testing, I used the Ubuntu 22.04.2 LTS (Jammy Jellyfish) 64-bit PC (AMD64) server install image (ubuntu-22.04.2-live-server-amd64.iso) from https://releases.ubuntu.com/jammy/. A good alternative might be the minimal x86_64 version of rocky linux https://rockylinux.org/download or alma linux https://mirrors.almalinux.org/isos.html (although, I am not sure about the compatibility of the package managers and installers when building your singularity file as rocky linux uses dnf as package manager).
3. Create a new VM using UTM: New VM -> Emulate -> Linux -> Select Image -> Select Memory and CPU -> Done.
4. Install your distribution on your new VM. Make sure to select the \*.iso as drive, and to unmount it after completing the installation. Probably you will not need most of the packages, so make sure to keep your installation light. Start your VM.

## Prerequisites
5. **Please note, that the following will not work using rocky linux, but must be adapted regarding the dnf-installer.**
6. Install prerequisites:
``` shell
sudo apt-get update && \
   sudo apt-get install -y \
      wget \
      build-essential \
      libseccomp-dev \
      libglib2.0-dev \
      pkg-config \
      squashfs-tools \
      cryptsetup \
      runc \
      curl
```
6. Install go. Use the most recent release by adapting the path (Reference: https://go.dev/dl/). Make sure to use the AMD64 version.
``` shell
curl -LO https://go.dev/dl/go1.20.2.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && \
   sudo tar -C /usr/local -xzf go1.20.2.linux-amd64.tar.gz
rm go1.20.2.linux-amd64.tar.gz
```
7. Make sure that the go binaries are present in bash’s PATH
``` shell
echo 'export PATH=/usr/local/go/bin:$PATH' >> ~/.bashrc && \
  source ~/.bashrc
```
8. Check whether go is ready to run: `go version` should print sth like `go version go1.20.2 linux/amd64`

----------------
## Singularity
For the installation of singularity, you have 2 options:

### Option A: compiled version
For some environments / OS, sylabs provides compiled releases [here](https://github.com/sylabs/singularity/releases). If so you can try to install it directly, which will save you the compiling.

9. Load compiled singularity files from [here](https://github.com/sylabs/singularity/releases). Make sure to use an appropriate version that matches your VM. In our case `singularity-ce_3.11.1-jammy_amd64.deb`.
``` shell
curl -LO https://github.com/sylabs/singularity/releases/download/v3.11.1/singularity-ce_3.11.1-jammy_amd64.deb
```
10. Install compiled version
``` shell
sudo dpkg -i singularity-ce_3.11.1-jammy_amd64.deb
```
*Note:* I faced an unspecific error during installation, but using `apt --fix-broken install` got rid of it.

### Option B: from source
Alternatively, and when you face problems using Option A, you can load the source and compile it at your machine.

9. Load compiled singularity files from [here](https://github.com/sylabs/singularity/releases). Make sure to use an appropriate version that matches your VM and HPC.
``` shell
curl -LO https://github.com/sylabs/singularity/releases/download/v3.11.1/singularity-ce-3.11.1.tar.gz && \
   tar -xf singularity-ce-3.11.1.tar.gz
rm singularity-ce-3.11.1.tar.gz
```
10. Compile and install singularity. This will take some time as compiling is really slow on your emulated VM:
``` shell
cd singularity-ce-3.11.1
./mconfig && \
    make -C builddir && \
    sudo make -C builddir install
```

----------------
Independently from the option you chose, continue here:

11. Make sure singularity was installed successfully: `singularity --version` should print sth like `singularity-ce version 3.11.1` or `3.11.1-jammy`
12. To test your singularity installation, run `singularity run library://lolcow`
13. To shutdown your VM: `shutdown -h now`.

## Mount shared folder
You can mount the folder you specified as shared in UTM to your VM:
1. Create a new directory, e.g. calles *shared-folder* `mkdir shared-folder`.
2. Second, mount the directory `sudo mount -t 9p -o trans=virtio share ~/shared-folder -oversion=9p2000.L`.
3. Third - to mount the directory at startup - modify the `/etc/fstab` file with `sudo nano` by adding:
```shell
share	~/shared-folder	9p	trans=virtio,version=9p2000.L,rw,_netdev,nofail	0	0
```

## SSH to VM
You can set up a ssh-connection to your VM from your host machine. This might be more convenient, than using the terminal in UMT.
1. Check whether ssh is installed, running and listening at your VM: `sudo systemctl status ssh`
2. Get the IP-adress of your VM at startup. Alternatively, use `ip -4 addr`, after `enp0s1 [...] inet _____` your VMs IP is shown.
3. You can connect from your host to your VM using `ssh <username>@<vm-ip-adress>`

Simplify your life by:

4. Optionally: Add the VM fingerprint to your `.ssh/known_host` manually: At your VM, go to `etc/ssh`, open the **public** key of the encryption algorithm you are going to use, e.g. `ssh_host_ed25519_key.pub`. At your host, go to `.ssh/known_hosts` and add the new host as a new line `<vm-ip-adress> <encryption-algorithm> <public-key-from-vm>`.
5. Create a new ssh-key at your host by `ssh-keygen -t ed25519`. I recommend to use a keyphrase and a specific name.
6. Store the **public** key at the VM: `ssh-copy-id -i ~/.ssh/<public-key-file>.pub <username>@<vm-ip-adress>`. If you did not add the host manually, you have to accept its fingerprint.
7. Add the VM to `.ssh/config` at your host, for instance:
```
Host vm_ubuntu
  HostName <vm-ip-adress>
  User <username>
  IdentityFile ~/.ssh/<private-key-file>
  #   UseKeychain <yes/no>
```
If you want to store the keyphrase in the keychain of your mac, use  `  UseKeychain yes`.

8. You can now connect from your host to your VM using `ssh vm_ubuntu`.
9. Optional: Deactivate ssh-authentification by password on your vm. Short version: In `etc/ssh/sshd_config` replace `#PasswordAuthentication yes` by `PasswordAuthentication no`. If there are any `/etc/ssh/sshd_config.d/*.conf`, make sure to replace (if existing) `PasswordAuthentication yes` by `PasswordAuthentication no`. Restart ssh / your vm. If you try to ssh to the VM without using your key-file, you should not even be asked for your password. More information are [here](https://help.ubuntu.com/community/SSH/OpenSSH/Configuring).

## Storage
Singularity is storage intensive, especially during the building-process due to temporary files. During installation of ubuntu, not all the (virtual) storage is allocated. Therefore, **if LVM was selected during installation**, extend it:
1. show all partitions using `sudo lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT,LABEL` or `fdisk -l`
2. identify the partition with ubuntu (e.g. *ubuntu--vg-ubuntu--lv*) and free storage. In my (and most) cases: `/dev/sda3`.
3. start parted with `sudo parted`, and use `resizepart`
4. indicate the partition you want to resize, in our case `3`
5. define the new size. If you want to add all free space to your partition, use `100%`
6. Exit parted with `quit` or Ctrl+C
7. Resize the partition with `pvresize`, e.g. `pvresize /dev/sda3`
8. Extend the volume with `lvextend`, e.g. `lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv`
9. Pass the changes to the filesystem with `resize2fs` , e.g. `resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv`
10. Check the changes using the commands described at 1.
   
## Notes
- I had some trouble regarding the inputs from the keyboard. For the tilde `~`, I had to press `option_right` + `+`.

## References
- https://docs.sylabs.io/guides/3.0/user-guide/installation.html
- https://sylabs.io/2023/03/installing-singularityce-on-macos-with-apple-silicon-using-utm/
- https://sylabs.io/2023/03/installing-singularityce-on-macos-with-apple-silicon-using-utm-rocky/
