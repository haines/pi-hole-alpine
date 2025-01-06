# Setting up Pi-hole in Docker on diskless Alpine Linux on a Raspberry Pi

My SD card is 32GB.
I chose to partition it with

- 256 MiB (FAT32) for the operating system,
- 2 GiB (ext4) for the local backup utility and package cache, and
- the remainder (ext4) for `/var/lib/docker`.

## Prepare the SD card (on macOS)

Find the device name with

```console
$ diskutil list
```

In my case, the SD card was `/dev/disk4`.

Partition the SD card with

```console
$ diskutil partitionDisk /dev/disk4 \
  MBR \
  FAT32 PI-HOLE 256MiB \
  FREE %noformat% R
```

Download and extract Alpine Linux to the SD card with

```console
$ gpg --receive-keys 293ACD0907D9495A
gpg: key 293ACD0907D9495A: public key "Natanael Copa <ncopa@alpinelinux.org>" imported

$ gpg --lsign-key 293ACD0907D9495A

$ curl --remote-name-all https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/aarch64/alpine-rpi-3.20.0-aarch64.tar.gz{.asc,}

$ gpg --verify alpine-rpi-3.20.0-aarch64.tar.gz{.asc,}
gpg: Signature made Wed 22 May 11:33:03 2024 BST
gpg:                using RSA key 0482D84022F52DF1C4E7CD43293ACD0907D9495A
gpg: Good signature from "Natanael Copa <ncopa@alpinelinux.org>" [full]

$ tar \
  --extract \
  --file alpine-rpi-3.20.0-aarch64.tar.gz \
  --directory /Volumes/PI-HOLE
```

Eject the SD card with

```console
$ diskutil eject /Volumes/PI-HOLE
```

## Generate an SSH key

Generate an SSH key in 1password with

```console
$ op item create --category ssh --title pihole
```

Upload the public key as a gist with

```console
$ op read "op://personal/pihole/public key" |
  gh gist create --filename pihole.pub -
https://gist.githubusercontent.com/...
```

Take a note of the URL for later.

## Set up Alpine

Insert the SD card and start the Raspberry Pi.
Login as `root` with no password.

Set up Alpine in diskless mode with

```console
# setup-alpine
```

| Setting | Value | Notes |
|---|---|---|
| Keyboard layout | `gb` |
| Keyboard layout variant | `gb` |
| System hostname | `pihole` |
| Network interface | `eth0` |
| IP address | `192.168.0.2` | This depends on the router's LAN settings |
| Netmask | `255.255.255.0` |
| Gateway | `192.168.0.1` |
| Manual network configuration | `n` |
| DNS domain name | (empty) |
| DNS nameservers | `1.1.1.1 1.0.0.1` |
| Timezone | `UTC` |
| HTTP/FTP proxy URL | `none` |
| NTP client | `chrony` |
| Mirror | `uk.alpinelinux.org` |
| Setup a user? | `pihole` |
| SSH server | `openssh` |
| Unmount `/media/mmcblk0p1`? | `n` |
| Where to store configs | `none` |
| `apk` cache directory | `/var/cache/apk` |

Install certificate authority certificates with

```console
# apk add ca-certificates
```

Enable the `community` repository and configure the mirror with HTTPS by running

```
# cat >/etc/apk/repositories <<EOF
/media/mmcblk0p1/apks
https://uk.alpinelinux.org/alpine/v3.20/main
https://uk.alpinelinux.org/alpine/v3.20/community
EOF
```

Upgrade any outdated packages with

```console
# apk update

# apk upgrade
```

Finish partitioning the SD card with

```console
# apk add e2fsprogs parted

# parted /dev/mmcblk0 mkpart primary ext4 257MiB 2305MiB

# mkfs.ext4 /dev/mmcblk0p2

# parted /dev/mmcblk0 mkpart primary ext4 2305MiB 100%

# mkfs.ext4 /dev/mmcblk0p3

# apk del e2fsprogs parted
```

Create mount points for the writable partitions with

```console
# mkdir /media/mmcblk0p2 /var/lib/docker
```

Add the writable partitions with

```
# cat >>/etc/fstab <<EOF
/dev/mmcblk0p2 /media/mmcblk0p2 ext4 defaults 0 0
/dev/mmcblk0p3 /var/lib/docker ext4 defaults 0 0
EOF
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

Authorize the SSH key with

```console
# su - pihole

$ mkdir ~/.ssh

$ chmod a=,u=rwx ~/.ssh

$ wget --output-document ~/.ssh/authorized_keys https://gist.githubusercontent.com/...

$ chmod a=,u=rw ~/.ssh/authorized_keys

$ exit
```

Allow `chronyd` to step the system clock with

```
# echo "makestep 1 -1" >>/etc/chrony/chrony.conf
```

Install Docker with

```console
# apk add docker

# rc-update add docker
```

Enable user namespace remapping with

```console
# echo "pihole:65536:65536" >/etc/subgid

# echo "pihole:65536:65536" >/etc/subuid

# mkdir /etc/docker

# echo '{"userns-remap":"pihole"}' >/etc/docker/daemon.json
```

Start Docker with

```console
# service docker start
```

Create a file setting the Pi-hole environment variables to

```
DNSMASQ_LISTENING=all
PIHOLE_DNS_=1.1.1.1;1.0.0.1
WEBPASSWORD=...
```

with

```console
# vi /etc/pihole

# chmod u=rw,g=r,o= /etc/pihole
```

Add a script to start Pi-hole with

```console
# cat >/usr/local/bin/pihole <<EOF
docker run \
  --detach \
  --dns 127.0.0.1 \
  --dns 1.1.1.1 \
  --dns 1.0.0.1 \
  --env-file /etc/pihole \
  --mount type=volume,source=dnsmasq,target=/etc/dnsmasq.d \
  --mount type=volume,source=pihole,target=/etc/pihole \
  --name pihole \
  --publish 53:53/tcp \
  --publish 53:53/udp \
  --publish 80:80 \
  --pull always \
  --restart always \
  pihole/pihole:latest
EOF

# chmod +x /usr/local/bin/pihole

# lbu include /usr/local/bin/pihole
```

Start Pi-hole with

```console
# pihole
```

Clear the message of the day with

```console
# : >/etc/motd
```

Commit configuration with

```console
# lbu commit
```
