# 📄 NAS Server Documentation & Configuration Summary (End-to-End)

This document details the complete setup of the server, covering LUKS Encryption, System Persistence, Linux Permissions, and the Docker Samba Service.

## 1. 🛡️ Server Setup & Disk Persistence

The initial phase ensured the physical disks were encrypted and configured to unlock automatically during boot (requiring the password only once).

A. Disk Preparation & Encryption

1. Encryption: Used `sudo cryptsetup luksFormat /dev/sdX` on the two raw disks (`/dev/sdb`, `/dev/sdc`).
2. Unlocking (Mapper Creation): Used `sudo cryptsetup luksOpen /dev/sdX <name>` to create the decrypted block devices under `/dev/mapper/` (e.g., `nas_disk1_crypt`).
3. Filesystem: Used `sudo mkfs.ext4 /dev/mapper/<name>` to format the unlocked devices.

B. LUKS Auto-Unlock (Persistence)

To make the drives unlockable at boot, the unique LUKS UUIDs were used:
1. /etc/crypttab: Configured to instruct the system to prompt for the LUKS password(s) early during boot and create the mapper devices. 
    - Example Entry: `nas_disk1_crypt UUID=<sdb_LUKS_UUID> none luks,discard`
2. /etc/fstab: Configured to automatically mount the decrypted mapper devices once they appear. 
    - Example Entry: `/dev/mapper/nas_disk1_crypt /srv/nas/disk1 ext4 defaults 0 2`
4. Initramfs Update: Ran `sudo update-initramfs -u` to ensure the boot environment includes the instructions from `/etc/crypttab`.

## 2. 👥 User & Permission Structure
This structure enforces the security policy for both local and network access.This structure enforces the security policy for both local and network access.

|Component       |User/Group                |Action Taken                                          |Permission Applied                                                                     |Resulting Access Policy                                                                   |
|----------------|--------------------------|------------------------------------------------------|---------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------|
|Admin User      |tinman (PUID: $SAMBA_PUID)|Primary account; added to nas_rw_group.               |N/A                                                                                    |Full control everywhere.                                                                  |
|Guest User      |guestro                   |Created with /usr/sbin/nologin; added to nas_rw_group.|N/A                                                                                    |R/W access to Disk 1 only (via group membership).                                         |
|Disk 1 (Shared) |/srv/nas/disk1            |Owner: tinman, Group: nas_rw_group.                   |2775                                                                                   |R/W for tinman and guestro. The SetGID bit (2) ensures files created inherit nas_rw_group.|
|Disk 2 (Private)|/srv/nas/disk2            |Owner: tinman, Group: tinman.                         |700                                                                                    |R/W for tinman ONLY. Denies all access to the group and others (including guestro).       |

## 3. 📦 Docker Samba Service (dperson/samba)
The final service was deployed using Docker Compose V2, leveraging the secure file system established above.

A. Configuration Summary
|Item            |Value                     |Significance                                          |
|----------------|--------------------------|------------------------------------------------------|
|Docker Compose  |V2 (docker compose)       |CRITICAL FIX: Required to avoid the KeyError: 'ContainerConfig' traceback caused by the legacy V1 executable.|
|Image           |dperson/samba:latest      |Selected as a reliable alternative to the GHCR image, avoiding authentication issues.|
|Network         |network_mode: "host"      |Used for maximum compatibility, allowing the container to expose SMB ports directly on the host's IP address.|
|Permissions     |USERID=$SAMBA_PUID, GROUPID=$SAMBA_PGID|Mapped PUID/PGID to ensure the Samba process respects the host's tinman:nas_rw_group ownership and permissions.|
|command:        |-u, -s, -p arguments      |Configures the Samba server inside the container, defining network users (tinman, guestro) and specifying access control for the two shares based on the users permitted by the host's file permissions.|

B. Access Logic via Samba Shares
|Share Name      |Host Path                 |Samba Access List                                     |Final Access Result                                                                    |
|----------------|--------------------------|------------------------------------------------------|---------------------------------------------------------------------------------------|
|shared_files    |/srv/nas/disk1            |tinman,guestro                                        |Both users can read and write (R/W).                                                   |
|private_admin   |/srv/nas/disk2            |tinman                                                |Only tinman can access (R/W). Access is denied to guestro by the host's 700 permission.|

## 4. 🚀 Execution & Next Steps

The service is launched using the V2 syntax, reading the configuration from the .env file:
```bash
docker compose up -d
```
## 5. 🔑 Environment Variables (.env File Content)
```bash
# --- System and User IDs (Mapped to dperson/samba USERID/GROUPID) ---
# These must match the IDs on your Ubuntu host system.

SAMBA_PUID=1000                  # Sample PUID for the 'tinman' user (Run: id -u tinman)
SAMBA_PGID=1002                  # Sample PGID for the 'nas_rw_group' (Run: id -g nas_rw_group)
TZ=Asia/Manila                   # Sample Timezone (e.g., America/New_York)

# --- Admin User Credentials ---

ADMIN_USER=tinman                # Your primary administrator username
ADMIN_PASS=ChangeThisPassword1   # Samba password for tinman

# --- Guest User Credentials ---

GUEST_USER=guestro               # Low-privilege username
GUEST_PASS=ChangeThisPassword2   # Samba password for guestro

# --- Host File Paths ---

HOST_DISK_1_PATH=/srv/nas/disk1  # Host path for the Shared Disk (LUKS mounted path)
HOST_DISK_2_PATH=/srv/nas/disk2  # Host path for the Private Disk (LUKS mounted path)

# --- Share 1 (Shared Files) Configuration ---

SHARE_1_NAME=shared_files        # The name of the Samba share visible on the network
SHARE_1_PATH=/mnt/disk1_shared   # Container path for Disk 1 (Must match volume map in YML)
SHARE_1_USERS=tinman,guestro     # Users allowed to access this share (comma-separated list)

# --- Share 2 (Private Admin) Configuration ---

SHARE_2_NAME=private_admin       # The name of the Samba share visible on the network
SHARE_2_PATH=/mnt/disk2_priv     # Container path for Disk 2 (Must match volume map in YML)
SHARE_2_USERS=tinman             # Users allowed to access this private share (Only tinman)
```
