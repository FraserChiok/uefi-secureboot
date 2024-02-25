# UEFI Secure Boot Documentation for Orin NX 16GB

## Prerequisites

Before starting, ensure that the following utilities are installed on your host machine:

- `openssl`
- `device-tree-compiler`
- `efitools`
- `uuid-runtime`

## Prepare the PK, KEK, db Keys

Generate the PK, KEK, and db RSA keypairs and certificates by following these steps:

1. Navigate to `<LDK_DIR>`
2. Create a directory named `uefi_keys`
3. Move to the `uefi_keys` directory

```bash
cd <LDK_DIR>
mkdir uefi_keys
cd uefi_keys
```

### Generate PK RSA keypair and certificate

```bash
openssl req -newkey rsa:2048 -nodes -keyout PK.key -new -x509 -sha256 -days 3650 -subj "/CN=my Platform Key/" -out PK.crt
```

### Generate KEK RSA keypair and certificate

```bash
openssl req -newkey rsa:2048 -nodes -keyout KEK.key -new -x509 -sha256 -days 3650 -subj "/CN=my Key Exchange Key/" -out KEK.crt
```

### Generate db_1 and db_2 RSA keypairs and certificates

```bash
openssl req -newkey rsa:2048 -nodes -keyout db_1.key -new -x509 -sha256 -days 3650 -subj "/CN=my Signature Database key/" -out db_1.crt
openssl req -newkey rsa:2048 -nodes -keyout db_2.key -new -x509 -sha256 -days 3650 -subj "/CN=my another Signature Database key/" -out db_2.crt
```

### Create a UEFI Keys Config File

Create a UEFI keys config file named `uefi_keys.conf` and insert the following lines:

```bash
UEFI_PK_KEY_FILE="PK.key";
UEFI_PK_CERT_FILE="PK.crt";
UEFI_KEK_KEY_FILE="KEK.key";
UEFI_KEK_CERT_FILE="KEK.crt";
UEFI_DB_1_KEY_FILE="db_1.key";
UEFI_DB_1_CERT_FILE="db_1.crt";
UEFI_DB_2_KEY_FILE="db_2.key";
UEFI_DB_2_CERT_FILE="db_2.crt";
```

## Generate Signed UEFI Payloads

All UEFI payloads must be signed using UEFI security keys. The following steps explain how to sign various UEFI payloads.

Note: Make sure the unsigned UEFI payloads are copied to the `uefi_keys/` folder before proceeding.

### Signing extlinux.conf

```bash
openssl cms -sign -signer db.crt -inkey db.key -binary -in extlinux.conf -outform der -out extlinux.conf.sig
```

### Signing initrd

```bash
openssl cms -sign -signer db.crt -inkey db.key -binary -in initrd -outform der -out initrd.sig
```

### Signing Image (Kernel) of rootfs

```bash
cp Image Image.unsigned
sbsign --key db.key --cert db.crt --output Image Image
```

### Signing kernel-dtb of rootfs

```bash
openssl cms -sign -signer db.crt -inkey db.key -binary -in kernel_tegra234-p3701-0004-p3737-0000.dtb -outform der -out kernel_t
```

### Signing boot.img of kernel partition

```bash
../bootloader/mkbootimg --kernel Image --ramdisk initrd --board <rootdev> --output boot.img --cmdline <cmdline_string>
cp boot.img boot.img.unsigned
openssl cms -sign -signer db.crt -inkey db.key -binary -in boot.img -outform der -out boot.img.sig
truncate -s %2048 boot.img
cat boot.img.sig >> boot.img
```

Note: Replace `<cmdline_string>` with appropriate command-line strings.

### Signing kernel-dtb of kernel-dtb partition

```bash
cp tegra234-p3509-a02+p3767-0000.dtb tegra234-p3509-a02+p3767-0000.dtb.unsigned
openssl cms -sign -signer db.crt -inkey db.key -binary -in tegra234-p3509-a02+p3767-0000.dtb -outform der -out tegra234-p3509-a02+p3767-0000.dtb.sig
truncate -s %2048 tegra234-p3509-a02+p3767-0000.dtb
cat tegra234-p3509-a02+p3767-0000.dtb.sig >> tegra234-p3509-a02+p3767-0000.dtb
```

### Signing recovery.img of recovery partition

```bash
../bootloader/mkbootimg --kernel Image --ramdisk ../bootloader/recovery.ramdisk --output recovery.img --cmdline <rec_cmdline_string>
cp recovery.img recovery.img.unsigned
openssl cms -sign -signer db.crt -inkey db.key -binary -in recovery.img -outform der -out recovery.img.sig
truncate -s %2048 recovery.img
cat recovery.img.sig >> recovery.img
```

Note: `<rec_cmdline_string>` should be replaced with appropriate command-line strings.

### Signing recovery kernel-dtb of recovery-dtb partition

```bash
cp tegra234-p3509-a02+p3767-0000.dtb.rec tegra234-p3509-a02+p3767-0000.rec.unsigned
openssl cms -sign -signer db.crt -inkey db.key -binary -in tegra234-p3509-a02+p3767-0000.dtb.rec -outform der -out tegra234-p3509-a02+p3767-0000.dtb.rec.sig
truncate -s %2048 tegra234-p3509-a02+p3767-0000.dtb.rec
cat tegra234-p3509-a02+p3767-0000.dtb.rec.sig >> tegra234-p3509-a02+p3767-0000.dtb.rec
```

### Signing BOOTAA64.efi

```bash
cp BOOTAA64.efi BOOTAA64.efi.unsigned
sbsign --key db.key --cert db.crt --output BOOTAA64.efi BOOTAA64.efi
```

## Enabling UEFI Secure Boot at Flashing Time

To enable UEFI Secure Boot at flashing time, use the `--uefi-keys <keys.conf>` option.

```bash
sudo ./tools/kernel_flash/l4t_initrd_flash.sh --external-device nvme0n1p1 -u <pkc_keyfile> [-v <sbk_keyfile>] --uefi-keys <keys.conf>
```

Note: Ensure that you replace `<pkc_keyfile>` with appropriate filenames.
