# Guide: Setting Up a Steam Library via iSCSI on NAS (Linux)

## Requirements

* NAS with iSCSI support (e.g., Synology, TrueNAS)
* DSM 7+ or equivalent with iSCSI configuration
* Linux system (e.g., Manjaro, Ubuntu, Arch)
* `open-iscsi` package installed
* Recommended filesystem: ext4
* Optional: Space Reclamation (`fstrim`) for Thin Provisioning

---

## Part 1: Set up iSCSI on your NAS (Example: Synology DSM)

### 1. Open SAN Manager or iSCSI Manager

* On the NAS web interface

### 2. Create an iSCSI Target

* Give it a name (e.g., `steam-target`)
* Enable CHAP authentication (e.g., username: `steamuser`, password: `SuperSecure123`)

### 3. Create a LUN

* Use block-level LUN
* Size: e.g., 200 GB to several TB
* Enable Thin Provisioning (recommended)
* Enable Space Reclamation
* Link the LUN to the iSCSI Target

---

## Part 2: Prepare Linux

### 1. Install open-iscsi package

```bash
sudo pacman -S open-iscsi        # Arch/Manjaro
sudo apt install open-iscsi      # Debian/Ubuntu
```

### 2. Start and enable the iSCSI service

```bash
sudo systemctl enable --now iscsid
```

### 3. Discover and configure the iSCSI Target with CHAP

```bash
sudo iscsiadm -m discovery -t sendtargets -p <NAS-IP>

# Set CHAP credentials
sudo iscsiadm -m node -T <iqn> -p <NAS-IP> --op=update -n node.session.auth.authmethod -v CHAP
sudo iscsiadm -m node -T <iqn> -p <NAS-IP> --op=update -n node.session.auth.username -v steamuser
sudo iscsiadm -m node -T <iqn> -p <NAS-IP> --op=update -n node.session.auth.password -v SuperSecure123

# Enable auto-login
sudo iscsiadm -m node -T <iqn> -p <NAS-IP> --op=update -n node.startup -v automatic

# Login to the target
sudo iscsiadm -m node -T <iqn> -p <NAS-IP> --login
```

---

## Part 3: Format and mount the iSCSI volume

### 1. Format the disk (only once)

```bash
sudo mkfs.ext4 /dev/sdX
```

### 2. Create mount point and mount

```bash
sudo mkdir -p /mnt/SteamNAS
sudo mount /dev/sdX /mnt/SteamNAS
sudo chown -R $USER:$USER /mnt/SteamNAS
```

---

## Part 4: Automount using systemd

### 1. Create systemd mount unit

```bash
sudo nano /etc/systemd/system/mnt-SteamNAS.mount
```

### Content:

```ini
[Unit]
Description=Mount iSCSI SteamNAS
Requires=iscsi.service
After=network-online.target iscsid.service iscsi.service

[Mount]
What=/dev/sdX
Where=/mnt/SteamNAS
Type=ext4
Options=defaults

[Install]
WantedBy=multi-user.target
```

### 2. Enable and start the mount unit

```bash
sudo systemctl daemon-reload
sudo systemctl enable mnt-SteamNAS.mount
sudo systemctl start mnt-SteamNAS.mount
```

---

## Part 5: Configure Steam

### 1. Create Steam Library folder

```bash
mkdir -p /mnt/SteamNAS/SteamLibrary
sudo chown -R $USER:$USER /mnt/SteamNAS/SteamLibrary
```

### 2. In Steam:

* Go to Settings > Storage > Add Library Folder
* Choose: `/mnt/SteamNAS/SteamLibrary`

---

## Part 6: Enable fstrim (recommended with Thin Provisioning)

```bash
sudo systemctl enable fstrim.timer
sudo systemctl start fstrim.timer
```

---

## ✅ Done

Your Steam library is now running from an iSCSI-mounted network disk with automatic mounting and space optimization. Platform-independent, performant, and future-ready!
