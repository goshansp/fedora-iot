# Scope
This project is aimed to deploy Fedora IoT to Rpi5. At the time of writing, mainline Kernel does not support Rpi5 networking. 


# Todo
- add rpi5 device tree, ...
- Include rpi5 ethernet network compatible Kernel (drowel)
- Verify if IoT is the only parent image that will boot with rpi
- Leverage Ignition for User commissioning
- clean up ansible_role_rpi_sensor


# Reference: How To Create SD Card for Rpi05 with official image (dupe from ansible_role_rpi_sensor)
```
wget https://download.fedoraproject.org/pub/alt/iot/42/IoT/aarch64/images/Fedora-IoT-raw-42-20250724.1.aarch64.raw.xz
$ sudo rpm-ostree install arm-image-installer --apply-live
# important: logout from any automounting gui sessions to avoid installation issues
# or: $ sudo umount /dev/sda3; sudo umount /dev/sda5; (this also unmounted /dev/sda)

# analysis
xz -d Fedora-IoT-raw-42-20250724.1.aarch64.raw.xz

$ fdisk -l ...
Device                                    Boot   Start     End Sectors  Size Id Type
Fedora-IoT-raw-42-20250724.1.aarch64.raw1 *      18432 1044479 1026048  501M  6 FAT16
Fedora-IoT-raw-42-20250724.1.aarch64.raw2      1044480 3141631 2097152    1G 83 Linux
Fedora-IoT-raw-42-20250724.1.aarch64.raw3      3141632 8402943 5261312  2.5G 83 Linux

```

## Prepare Rpi5 as Build env / Classic Fedora 42
```
$ sudo dnf install git ostree rpm-ostree composer-cli osbuild-composer
$ sudo systemctl enable osbuild-composer.socket && sudo systemctl start osbuild-composer.socket
```


## Step 1: Compose the Ostree Filesystem (on aarch64)
```
$ git clone -b "f42" https://pagure.io/fedora-iot/ostree.git
$ cd ostree
$ ostree init --repo=repo --mode=archive
$ cp ../fedora-iot-rpi5.yaml .; cp ../kernel-rpi.repo .

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

## Step 3: Deploy Rpi5 Devicetree
Rpi5 is not supported yet by the mainline kernel and hence we need to hack the devicetree file into proper path.
```
$ mkdir sda1; mkdir sda2
$ sudo mount /dev/sda1 sda1; sudo mount /dev/sda2 sda2
$ sudo cp sda2/ostree/fedora-iot-8a08b76006845deb2c0376d33b936a66cf957865ff8e86d4ccb451563c31132d/dtb/broadcom/bcm2712-rpi-5-b.dtb sda1/.
$ sudo umount /dev/sda*
```

## Step 4: Kernel for Rpi Network


## Step 5: Flash the image to the SD Card
```
$ mount | grep sda
$ umount /dev/sda*

$ sudo arm-image-installer --image=0ddbc300-a30d-44a0-b579-eace5b30db25-image.raw.xz --media=/dev/sda --resizefs --addkey /home/hp/.ssh/id_ecdsa.pub

```