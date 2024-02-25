Prerequisites
Ensure that the following utilities are installed in your host:
openssl

device-tree-compiler

efitools

uuid-runtime

Prepare the PK, KEK, db Keys
Generate the PK, KEK, db RSA keypairs and certificates
To generate the PK, KEK, and db RSA keypairs and certificates, run the following commands:

$ cd to <LDK_DIR>
$ mkdir uefi_keys
$ cd uefi_keys

### Generate PK RSA keypair and certificate
$ openssl req -newkey rsa:2048 -nodes -keyout PK.key  -new -x509 -sha256 -days 3650 -subj "/CN=my Platform Key/" -out PK.crt

### Generate KEK RSA keypair and certificate
$ openssl req -newkey rsa:2048 -nodes -keyout KEK.key  -new -x509 -sha256 -days 3650 -subj "/CN=my Key Exchange Key/" -out KEK.crt

### Generate db_1 RSA keypair and certificate
$ openssl req -newkey rsa:2048 -nodes -keyout db_1.key  -new -x509 -sha256 -days 3650 -subj "/CN=my Signature Database key/" -out db_1.crt

### Generate db_2 RSA keypair and certificate
$ openssl req -newkey rsa:2048 -nodes -keyout db_2.key  -new -x509 -sha256 -days 3650 -subj "/CN=my another Signature Database key/" -out db_2.crt

Create a UEFI Keys Config File
To create a UEFI keys config file with the generated keys, run the following command:

$ vim uefi_keys.conf
Insert the following lines to uefi_keys.conf file:

UEFI_PK_KEY_FILE="PK.key";
UEFI_PK_CERT_FILE="PK.crt";
UEFI_KEK_KEY_FILE="KEK.key";
UEFI_KEK_CERT_FILE="KEK.crt";
UEFI_DB_1_KEY_FILE="db_1.key";
UEFI_DB_1_CERT_FILE="db_1.crt";
UEFI_DB_2_KEY_FILE="db_2.key";
UEFI_DB_2_CERT_FILE="db_2.crt";

Generate Signed UEFI Payloads
All UEFI payloads have to be signed using UEFI security keys. If the –uefi-keys option is specified during flashing, the UEFI payloads are signed automatically by the flash.sh script. To enable UEFI Secureboot at runtime from Ubuntu prompt, the UEFI payloads have to be signed from the host and then you can download the signed payloads to target.

The UEFI payloads are:
extlinux.conf

initrd

kernel images (in rootfs, and in kernel and recovery partitions)

kernel-dtb images (in rootfs, and in kernel-dtb and recovery-dtb partitions), and

BOOTAA64.efi.

Note

The following steps assume that you have copied the required unsigned UEFI payloads to the uefi_keys/ folder. Also, db.crt and db.key can be replaced with db_1.* or db_2.* key.

To sign extlinux.conf using db:

$ openssl cms -sign -signer db.crt -inkey db.key -binary -in extlinux.conf -outform der -out extlinux.conf.sig
To sign initrd using db:

$ openssl cms -sign -signer db.crt -inkey db.key -binary -in initrd -outform der -out initrd.sig
To sign Image (the kernel) of rootfs using db:

$ cp Image Image.unsigned
$ sbsign --key db.key --cert db.crt --output Image Image
To sign kernel-dtb of rootfs using db:

$ openssl cms -sign -signer db.crt -inkey db.key -binary -in kernel_tegra234-p3701-0004-p3737-0000.dtb -outform der -out kernel_t

To sign boot.img of kernel partition using db:

$ ../bootloader/mkbootimg --kernel Image --ramdisk initrd --board <rootdev> --output boot.img --cmdline <cmdline_string>
$ cp boot.img boot.img.unsigned
$ openssl cms -sign -signer db.crt -inkey db.key -binary -in boot.img -outform der -out boot.img.sig
$ truncate -s %2048 boot.img
$ cat boot.img.sig >> boot.img

where <cmdline_string>, when generated in flash.sh to flash eMMC/SD, is:
Orin Series:

root=/dev/mmcblk0p1 rw rootwait rootfstype=ext4 mminit_loglevel=4 console=ttyTCU0,115200 console=ttyAMA0,115200 
and the <cmdline_string>, when generated in l4t_initrd_flash.sh to flash NVMe, is:
root=/dev/nvme0n1p1 rw rootwait rootfstype=ext4 mminit_loglevel=4 console=ttyTCU0,115200 console=ttyAMA0,115200 firmware_class.path=/etc/firmware fbcon=map:0 net.ifnames=0

To sign kernel-dtb of kernel-dtb partition using db:

$ cp tegra234-p3509-a02+p3767-0000.dtb tegra234-p3509-a02+p3767-0000.dtb.unsigned
$ openssl cms -sign -signer db.crt -inkey db.key -binary -in tegra234-p3509-a02+p3767-0000.dtb -outform der -out tegra234-p3509-a02+p3767-0000.dtb.sig
$ truncate -s %2048 tegra234-p3509-a02+p3767-0000.dtb
$ cat tegra234-p3509-a02+p3767-0000.dtb.sig >> tegra234-p3509-a02+p3767-0000.dtb

To sign recovery.img of recovery partition using db:

$ ../bootloader/mkbootimg --kernel Image --ramdisk ../bootloader/recovery.ramdisk --output recovery.img --cmdline <rec_cmdline_string>
$ cp recovery.img recovery.img.unsigned
$ openssl cms -sign -signer db.crt -inkey db.key -binary -in recovery.img -outform der -out recovery.img.sig
$ truncate -s %2048 recovery.img
$ cat recovery.img.sig >> recovery.img

where <rec_cmdline_string> is:
Orin Series:
“root=/dev/initrd rw rootwait mminit_loglevel=4 console=ttyTCU0,115200 firmware_class.path=/etc/firmware fbcon=map:0 net.ifnames=0”

The Image inside recovery.img must also be signed. Use the Image signed by the step 3 above.

To sign recovery kernel-dtb of recovery-dtb partition using db:

$ cp tegra234-p3509-a02+p3767-0000.dtb.rec tegra234-p3509-a02+p3767-0000.rec.unsigned
$ openssl cms -sign -signer db.crt -inkey db.key -binary -in tegra234-p3509-a02+p3767-0000.dtb.rec -outform der -out tegra234-p3509-a02+p3767-0000.dtb.rec.sig
$ truncate -s %2048 tegra234-p3509-a02+p3767-0000.dtb.rec
$ cat tegra234-p3509-a02+p3767-0000.dtb.rec.sig >> tegra234-p3509-a02+p3767-0000.dtb.rec

To sign BOOTAA64.efi using db:

$ cp BOOTAA64.efi BOOTAA64.efi.unsigned
$ sbsign --key db.key --cert db.crt --output BOOTAA64.efi BOOTAA64.efi

Enabling UEFI Secureboot at Flashing Time
Using option –uefi-keys <keys_conf> to Provide Signing Keys and Enabling UEFI Secure Boot
Note

Although UEFI secure boot can be independently enabled from a low-level bootloader secure boot, we strongly recommended that users enable bootloader secure boot so that the root-of-trust can start from the BootROM.

Issue the following commands with --uefi-keys <keys.conf> option:

For the Jetson Orin NX series and the Orin Nano series:

$ sudo ./tools/kernel_flash/l4t_initrd_flash.sh --external-device nvme0n1p1 -u <pkc_keyfile> [-v <sbk_keyfile>] --ue

