Follow these instructions to run singularity on macOS Ventura @arm64. Solutions using VirtualBox did not work for me, as VirtualBox is available for the arm64-architecture but the vm-box provided by sylabs *sylabs/singularity-2.6-ubuntu-bionic64* crashed. Also alternative ubuntu-distributions do often not work either using arm64/AppleSilicon. Instead, we can establish a lightweight solution without dozens of packages and addons using multipass (see also https://github.com/canonical/multipass):

1. Install mulipass using HomeBrew or the installer: https://multipass.run/docs/installing-on-macos
2. Create a new multipass instance by @host:`multipass launch -n singularity focal` (name: singularity; version: focal = ubuntu 20.04). tbh, i think the version does not really matter, however, i would just try to mirror your target system as good as possible. For other versions or information regarding multipass see https://github.com/canonical/multipass).
3. Check whether your instance was created successfully: `multipass list`
4. Stop your instance: `multipass stop singularity`
5. More power to your instance: `multipass set local.singularity.cpus=4`, `multipass set local.singularity.disk=20G`, `multipass set local.singularity.memory=10G`. This can also be adapted later on (only after stopping your instance), and should depend on your host system. I would not exceed 50% of the host system. If you want to check your current settings use `multipass get local.singularity.cpus` / `.disk` / `.memory`
6. Login to your instance: `multipass shell singularity`. It takes some seconds, but you should then get some information regarding your instance and the system information.
7. To install singularity, you need a C compiler on your instance. Thus, first update the package index @vm:`sudo apt-get update`. Sedond install essentials **@vm**:`sudo apt install build-essential`.
8. Check whether you got it: **@vm**:`gcc --version` should print something line "gcc (Ubuntu 9.4.0-1ubuntu1~20.04.1) 9.4.0"
9. Follow the instructions above. For me worked **@vm**:
```shell
git clone https://github.com/sylabs/singularity.git
cd singularity
git fetch --all
git checkout v3.9.7
sudo snap install go --classic
./mconfig
cd builddir
make
sudo make install
```
10. Use **singularity** at your instance.
11. Exit with `exit`, stop your instance with `multipass stop singularity`
12. If not needed anymore: Kill your instance with `multipass delete singularity` and remove remaining files with `multipass purge`
