# Setting up Pi-hole in Docker on diskless Alpine Linux on a Raspberry Pi

My SD card is 16GB.
I chose to partition it with

- 192 MiB (FAT32) for the operating system,
- 1 GiB (ext4) for the local backup utility, and
- the remainder (ext4) for `/var/lib/docker`.

## Prepare the SD card (on macOS)

Find the device name with

```console
$ diskutil list
```

In my case, the SD card was `/dev/disk2`.

Partition the SD card with

```console
$ diskutil partitionDisk /dev/disk2 \
  MBR \
  FAT32 PI-HOLE 192MiB \
  FREE %noformat% R
```

Download and extract Alpine Linux to the SD card with

```console
$ gpg --receive-keys 293ACD0907D9495A
gpg: key 293ACD0907D9495A: public key "Natanael Copa <ncopa@alpinelinux.org>" imported

$ gpg --lsign-key 293ACD0907D9495A

$ curl --remote-name-all https://dl-cdn.alpinelinux.org/alpine/v3.13/releases/aarch64/alpine-rpi-3.13.1-aarch64.tar.gz{.asc,}

$ gpg --verify alpine-rpi-3.13.1-aarch64.tar.gz{.asc,}
gpg: Signature made Thu 11 Feb 15:16:57 2021 GMT
gpg:                using RSA key 0482D84022F52DF1C4E7CD43293ACD0907D9495A
gpg: Good signature from "Natanael Copa <ncopa@alpinelinux.org>" [full]

$ tar \
  --extract \
  --file alpine-rpi-3.13.1-aarch64.tar.gz \
  --directory /Volumes/PI-HOLE
```

Eject the SD card with

```console
$ diskutil eject /Volumes/PI-HOLE
```
