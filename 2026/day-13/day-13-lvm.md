# LINUX VOLUME MANAGEMENT.

## Switching to root user.

`sudo su`

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/fe6332abdc26867befcd002cb353eebe0f208e89/2026/day-13/Screenshots/Screenshot%20(502).png)

## Creating a virtual disk.

`dd if=/dev/zero of=/tmp/disk1.img bs=1M count=1024`
    
`losetup -fP /tmp/disk1.img`

`losetup -a`   # Note the device name (e.g., /dev/loop3)

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/fe6332abdc26867befcd002cb353eebe0f208e89/2026/day-13/Screenshots/Screenshot%20(503).png)

## Task 1: Checking Current Storage.

`lsblk`

`pvs`

`vgs`

`lvs`

`df -h`

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/fe6332abdc26867befcd002cb353eebe0f208e89/2026/day-13/Screenshots/Screenshot%20(504).png)

## Task 2: Creating Physical Volume.

`pvcreate /dev/loop3`

`pvs`

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/fe6332abdc26867befcd002cb353eebe0f208e89/2026/day-13/Screenshots/Screenshot%20(505).png)

## Task 3: Creating Volume Group.

`vgcreate devops-vg /dev/loop3`

`vgs`

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/fe6332abdc26867befcd002cb353eebe0f208e89/2026/day-13/Screenshots/Screenshot%20(506).png)

## Task 4: Creating Logical Volume.

`lvcreate -L 500M -n app-data devops-vg`

`lvs`

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/fe6332abdc26867befcd002cb353eebe0f208e89/2026/day-13/Screenshots/Screenshot%20(507).png)

## Task 5: Formating and Mounting.

`mkfs.ext4 /dev/devops-vg/app-data`

`mkdir -p /mnt/app-data`

`mount /dev/devops-vg/app-data /mnt/app-data`

`df -h /mnt/app-data`

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/fe6332abdc26867befcd002cb353eebe0f208e89/2026/day-13/Screenshots/Screenshot%20(508).png)

## Task 6: Extending the Volume.

`lvextend -L +200M /dev/devops-vg/app-data`

`resize2fs /dev/devops-vg/app-data`

`df -h /mnt/app-data`

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/fe6332abdc26867befcd002cb353eebe0f208e89/2026/day-13/Screenshots/Screenshot%20(509).png)

## What I Learned
1. Physical Volumes are the raw storage devices prepared for LVM.

2. Volume Groups pool multiple physical volumes into a single storage unit, enabling flexible allocation.

3. Logical Volumes act like partitions but can be resized dynamically, making storage management far more adaptable than traditional partitioning.

