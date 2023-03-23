# Singularity x86_64 @arm64
For some reasons it might be necessary to emulate a x86_64 architecture. For instance, building a singularity file in sandbox-mode is usually not possible on the HPC-cluster directly. The problem with the multipass solution is, that its ubuntu is an arm64-adapted version. Thus, your container will either work on your local machine, or on the cluster. To overcome this, emulating the x86_64 is necessary (but comes with limited performance):

1. We run the emulation using the open-source software QEMU. UTM is a much more comfortable solution based on QEMU to get started. Therefore, install UTM from https://mac.getutm.app.
2. We need an image. The version and kind is up to you, just make sure to choose a AMD64 image. For testing, I used the Ubuntu 20.04.6 LTS (Focal Fossa) 64-bit PC (AMD64) server install image (ubuntu-20.04.6-live-server-amd64.iso) from https://releases.ubuntu.com/focal/. A good alternative might be the minimal x86_64 version of rocky linux https://rockylinux.org/download or alma linux https://mirrors.almalinux.org/isos.html (although, I am not sure about the compatibility of the package managers and installers when building your singularity file as rocky linux uses dnf as package manager).
3. Create a new VM using UTM: New VM -> Emulate -> Linux -> Select Image -> Select Memory and CPU -> Done.
4. Install your distribution on your new VM. Make sure to select the \*.iso as drive, and to unmount it after completing the installation. Probably you will not need most of the packages, so make sure to keep your installation light. Start your VM.
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
   sudo tar -C /usr/local -xzf go1.20.2.linux-amd4.tar.gz
rm go1.20.2.linux-amd4.tar.gz
```
7. Make sure that the go binaries are present in bash’s PATH
``` shell
echo 'export PATH=/usr/local/go/bin:$PATH' >> ~/.bashrc && \
  source ~/.bashrc
```
8. Check whether go is ready to run: `go version` should print sth like `go version go1.20.2 linux/amd64`
9. Load singularity files. Make sure to use a appropriate version (either the most recent: https://github.com/sylabs/singularity/releases, or the version that is used at your HPC):
``` shell
curl -LO https://github.com/sylabs/singularity/releases/download/v3.11.1/singularity-ce-3.11.1.tar.gz && \
   tar -xf singularity-ce-3.11.1.tar.gz
rm singularity-ce-3.11.1.tar.gz
```
10. Install singularity. This will take some time as compiling is really slow on your emulated VM:
``` shell
cd singularity-ce-3.11.1
./mconfig && \
    make -C builddir && \
    sudo make -C builddir install
```
11. Make sure singularity was installed successfully: `singularity --version` should print sth like `singularity-ce version 3.11.1`
12. Mount the folder you specified in UTM to your VM: Therefore, first create a new directory `mkdir <shared-folder>`.
13. Second, mount the directory `sudo mount -t 9p -o trans=virtio share <shared-folder> -oversion=9p2000.L`.
14. Third - to mount the directory at startup - modify the `/etc/fstab` file with `sudo nano` and add `share	<shared-folder>	9p	trans=virtio,version=9p2000.L,rw,_netdev,nofail	0	0` to the file. Save your changes.
15. To shutdown your VM: `shutdown -h now`.

## Notes
- I had some trouble regarding the inputs. For the tilde ~, I had to press shift + option_left + #.

## References
https://sylabs.io/2023/03/installing-singularityce-on-macos-with-apple-silicon-using-utm/
https://sylabs.io/2023/03/installing-singularityce-on-macos-with-apple-silicon-using-utm-rocky/
