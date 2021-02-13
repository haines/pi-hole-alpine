# Setting up Pi-hole in Docker on diskless Alpine Linux on a Raspberry Pi

My SD card is 16GB.
I chose to partition it with

- 192 MiB (FAT32) for the operating system,
- 1 GiB (ext4) for the local backup utility and package cache, and
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

## Set up Alpine (on the Raspberry Pi)

Insert the SD card and start the Raspberry Pi.
Login as `root` with no password.

Set up Alpine in diskless mode with

```console
# setup-alpine
```

| Setting | Value | Notes |
|---|---|---|
| Keyboard layout | `gb` |
| Keyboard layout variant | `gb-mac` | I was using an Apple Magic Keyboard |
| System hostname | `pi-hole` |
| Network interface | `eth0` |
| IP address | `10.134.1.53` | This depends on the router's LAN settings |
| Netmask | `255.255.255.0` |
| Gateway | `10.134.1.1` |
| Manual network configuration | `n` |
| DNS domain name | (empty) |
| DNS nameservers | `1.1.1.1 1.0.0.1` |
| Timezone | `UTC` |
| HTTP/FTP proxy URL | `none` |
| NTP client | `chrony` |
| Mirror | `uk.alpinelinux.org` |
| SSH server | `openssh` |
| Unmount `/media/mmcblk0p1`? | `n` |
| Where to store configs | `none` |
| `apk` cache directory | `/var/cache/apk` |

Install certificate authorities with

```console
# apk add ca-certificates
```

Edit the `apk` repositories with

```console
# vi /etc/apk/repositories
```

Enable the `community` repository and configure the mirror with HTTPS:

```
/media/mmcblk0p1/apks
https://uk.alpinelinux.org/alpine/v3.13/main
https://uk.alpinelinux.org/alpine/v3.13/community
```

Upgrade any outdated packages with

```console
# apk update

# apk upgrade
```

Finish partitioning the SD card with

```console
# apk add e2fsprogs parted

# parted /dev/mmcblk0 mkpart primary ext4 193MiB 1217MiB

# mkfs.ext4 /dev/mmcblk0p2

# parted /dev/mmcblk0 mkpart primary ext4 1217MiB 100%

# mkfs.ext4 /dev/mmcblk0p3

# apk del e2fsprogs parted
```

Create mount points for the writable partitions with

```console
# mkdir /media/mmcblk0p2 /var/lib/docker
```

Edit the file systems table with

```console
# vi /etc/fstab
```

Add the writable partitions:

```
/dev/mmcblk0p2 /media/mmcblk0p2 ext4 defaults 0 0
/dev/mmcblk0p3 /var/lib/docker ext4 defaults 0 0
```

Mount the writable partitions with

```console
# mount -a
```

Set up the local backup utility with

```console
# setup-lbu mmcblk0p2
```

Set up the `apk` cache with

```console
# setup-apkcache /media/mmcblk0p2/cache
```

Add an unprivileged user with

```console
# adduser pi-hole
```

Clear the message of the day with

```console
# : >/etc/motd
```

Commit configuration with

```console
# lbu commit
```
