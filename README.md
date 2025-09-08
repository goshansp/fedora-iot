# Scope
This project is aimed to deploy Fedora IoT to Rpi5. At the time of writing, mainline Kernel does not support Rpi5 networking. Hence we need to fix `kernel-rpi` to bake into the image.


# Next
Is it possible to inject the copr/dwrobel-kernel into /boot? ... how can we achieve that?


# Todo
- add rpi5 device tree, ...
- Include rpi5 ethernet network compatible Kernel (drowel)
- why no usb keyboard
- 
- Leverage Ignition for User commissioning
- clean up ansible_role_rpi_sensor


## Step -3: Prepare Rpi5 as Build env / Classic Fedora 42
```
$ sudo dnf install git ostree rpm-ostree composer-cli osbuild-composer
$ sudo systemctl enable osbuild-composer.socket && sudo systemctl start osbuild-composer.socket
```

## Step -2: Compile Kernel-Rpi and create local repo
Build time on x86 aborted after 167 minutes. On Rpi5 it took 63 minutes.
```
$ git clone git@github.com:goshansp/kernel-rpi.git
$ toolbox enter
$ dnf install mock
$ mkdir ~/rpmbuild; mkdir ~/rpmbuild/SOURCES/
$ cp * ~/rpmbuild/SOURCES/. -r
$ curl -L https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.12.tar.xz -o ~/rpmbuild/SOURCES/linux-6.12.tar.xz
$ curl -L https://cdn.kernel.org/pub/linux/kernel/v6.x/patch-6.12.42.xz -o ~/rpmbuild/SOURCES/patch-6.12.42.xz
$ sudo usermod -aG mock hp

# aarch64 (full) - 63 minutes
mock -r fedora-42-aarch64 \
     --spec ~/rpmbuild/SOURCES/kernel.spec \
     --sources ~/rpmbuild/SOURCES/ \
     --resultdir ~/rpmbuild/RPMS \
     --buildsrpm --rebuild \
     --isolation=simple

# Rebuild Only?
mock -r fedora-42-aarch64 \
    --spec ~/rpmbuild/SOURCES/kernel.spec \
    --sources ~/rpmbuild/SOURCES/ \
    --resultdir ~/rpmbuild/RPMS \
    --rebuild ~/rpmbuild/RPMS/kernel-6.12.42-1.rpi.fc42.src.rpm

# x86
mock -r fedora-42-aarch64 \
     --spec ~/rpmbuild/SOURCES/kernel.spec \
     --sources ~/rpmbuild/SOURCES/ \
     --resultdir ~/rpmbuild/RPMS \
     --define "_smp_mflags -j$(nproc)" \
     --define "_memusage_limit 8192" \
     --buildsrpm --rebuild

$ ls /home/hp/rpmbuild/RPMS

$ createrepo ~/rpmbuild/RPMS

# the below two lines before the DO IT section ... on the kernel.spec
mkdir -p %{buildroot}/usr/lib/modules/$KernelVer
cp -a %{buildroot}/lib/modules/$KernelVer/* %{buildroot}/usr/lib/modules/$KernelVer/

or could we deploy 6.17-rc4? if so, how? can we just deploy a fedora 44 iot image to rpi?

```

## Step -1: Establish Ostree Compatibility for Kernel-Rpi
During `ostree compose tree` we're hitting an empty `/usr/lib/modules` that needs population from `/lib/modules`.
```
error: Postprocessing and committing: Finalizing rootfs: During kernel processing: /usr/lib/modules is empty


```

## Step 1: Compose the Ostree Filesystem (on aarch64)
Treefile Reference: https://coreos.github.io/rpm-ostree/treefile/
```
$ git clone -b "f42" https://pagure.io/fedora-iot/ostree.git
$ cd ostree
$ ostree init --repo=repo --mode=archive
$ cp ../fedora-iot/fedora-iot-rpi5.yaml ../fedora-iot/kernel-rpi.local.repo .

$ sudo rpm-ostree compose tree --repo=./repo --unified-core fedora-iot-rpi5.yaml

$ python3 -m http.server 8081
```

## Step 2: Compose bootable Disk Image for Rpi5
Information about `iot-raw-xz` can be found here https://osbuild.org/docs/user-guide/image-descriptions/fedora-42/iot-raw-xz/
```
$ sudo composer-cli blueprints push fedora-iot-rpi5.toml
$ sudo composer-cli blueprints depsolve custom-fiot-rpi5

$ sudo composer-cli compose start-ostree custom-fiot-rpi5 iot-raw-xz \
    --ref fedora/stable/aarch64/iot \
    --url http://127.0.0.1:8081/repo

$ sudo composer-cli compose list
ID                                     Status    Blueprint          Version   Type
0ddbc300-a30d-44a0-b579-eace5b30db25   RUNNING   custom-fiot-rpi5   0.0.1     iot-raw-xz

... wait for FINISHED.

$ sudo composer-cli compose image 0ddbc300-a30d-44a0-b579-eace5b30db25
```

## Step 3: Kernel for Rpi Network
Done in step 1.

## Step 4: Flash the image to the SD Card
```
$ mount | grep sda
$ umount /dev/sda*

$ sudo arm-image-installer --image=0ddbc300-a30d-44a0-b579-eace5b30db25-image.raw.xz --media=/dev/sda --resizefs

```

## Step 5: Deploy Rpi5 Devicetree to sd card
Rpi5 is not supported yet by the mainline kernel and hence we need to hack the devicetree file into proper path.
```
$ mkdir sda1; mkdir sda2
$ sudo mount /dev/sda1 sda1; sudo mount /dev/sda2 sda2
$ sudo cp sda2/ostree/fedora-iot-8a08b76006845deb2c0376d33b936a66cf957865ff8e86d4ccb451563c31132d/dtb/broadcom/bcm2712-rpi-5-b.dtb sda1/.
$ sudo umount /dev/sda*
```


# Reference: How To Create SD Card for Rpi3&4 with official image (dupe from ansible_role_rpi_sensor)
```
wget https://download.fedoraproject.org/pub/alt/iot/42/IoT/aarch64/images/Fedora-IoT-raw-42-20250724.1.aarch64.raw.xz
$ sudo rpm-ostree install arm-image-installer --apply-live
# important: logout from any automounting gui sessions to avoid installation issues
# or: $ sudo umount /dev/sda3; sudo umount /dev/sda5; (this also unmounted /dev/sda)

# analysis
xz -d Fedora-IoT-raw-42-20250724.1.aarch64.raw.xz

$ fdisk -l Fedora-IoT-raw-42-20250724.1.aarch64.raw
Device                                    Boot   Start     End Sectors  Size Id Type
Fedora-IoT-raw-42-20250724.1.aarch64.raw1 *      18432 1044479 1026048  501M  6 FAT16
Fedora-IoT-raw-42-20250724.1.aarch64.raw2      1044480 3141631 2097152    1G 83 Linux
Fedora-IoT-raw-42-20250724.1.aarch64.raw3      3141632 8402943 5261312  2.5G 83 Linux

$ sudo kpartx -av 7d5369d1-2f19-4cbf-8d47-4121d8d71e7e-image.raw
$ mkdir mnt_sda2
$ sudo mount /dev/mapper/loop0p2 mnt_sda2/
...
$ sudo umount /dev/mapper/loop0p*

$ sudo arm-image-installer --image=Fedora-IoT-raw-42-20250724.1.aarch64.raw.xz --media=/dev/sda --resizefs --addkey /home/hp/.ssh/id_ecdsa.pub

```


# ADR

## No Bootc
Bootc cannot apparently create 'iot-raw-xz' and 'composer' cannot feed off container images. Hence we stick with the old process until bootc achieves feature parity (rpi compatibility with the fat partition)?