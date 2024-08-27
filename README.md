# required manual setup

The following things need to be done manually just once

## ssh keys

run the ssh-keys.yml playbook (see its contents for more info)

## update the pis firmware

ssh onto the pi and
- append `cgroup_memory=1 cgroup_enable=memory` to /boot/firmware/cmdline.txt
- run: `rpi-update`
- then `reboot`

## drbd

on primary and replica do:

0. isntall drbd: `apt-get install drbd-utils`
1. plug in the hdd
2. create a partition on the hdd:
    - delete any existing partition with `fdisk /dev/sda`
        and enter 'd' then 'w' to apply changes
    - create partition on device
        `parted --script /dev/sda mklabel gpt mkpart data ext4 0% 100%`
3. get the partuuid and change it in the r0.res file underneath: `lsblk -o NAME,PARTUUID`
4. copy this file into /etc/drbd.d/r0.res  (with the correct partuuids from above)
    ```
    resource r0 {
        net {
            protocol A;
        }
        on k3s-master {
            address 192.168.2.130:7788;
            volume 0 {
                device /dev/drbd0;
                disk /dev/disk/by-partuuid/e50f45b8-2ce0-4cbe-adf4-f5c5eefae2b1;
                meta-disk internal;
            }
        }
        on k3s-node1 {
            address 192.168.2.204:7788;
            volume 0 {
                device /dev/drbd0;
                disk /dev/disk/by-partuuid/cecc1815-ad7e-48c6-a7cb-c16454f27e35;
                meta-disk internal;
            }
        }
    }
    ```
5. setup drbd
    ```
    drbdadm create-md r0
    drbdadm up r0
    systemctl enable drbd
    ``` 

Then on the master:
- create the data directory: `mkdir /data`
- create scripts directory `mkdir /root/scripts`
- create auto mount script `nano /root/scripts/mount.sh`:
    ```shell
    #!/bin/bash
    # Script to promote DRBD resource to primary and mount the device

    LOGFILE="/var/log/drbd-promote.log"

    echo "$(date): Starting DRBD promotion and mount script" >> $LOGFILE

    # Promote DRBD resource to primary
        if ! drbdadm status r0 | grep -q "Primary"; then
            echo "$(date): Promoting DRBD resource r0 to primary" >> $LOGFILE
            drbdadm primary --force r0 >> $LOGFILE 2>&1
            if [ $? -eq 0 ]; then
                echo "$(date): Successfully promoted DRBD resource r0 to primary" >> $LOGFILE
            else
                echo "$(date): Failed to promote DRBD resource r0 to primary" >> $LOGFILE
                exit 1
            fi
        else
        echo "$(date): DRBD resource r0 is already primary" >> $LOGFILE
    fi

    # Mount the DRBD device to /data
    if mountpoint -q /data; then
        echo "$(date): /data is already mounted" >> $LOGFILE
    else
        echo "$(date): Mounting /dev/drbd0 to /data" >> $LOGFILE
        mount -t ext4 /dev/drbd0 /data >> $LOGFILE 2>&1
        if [ $? -eq 0 ]; then
            echo "$(date): Successfully mounted /dev/drbd0 to /data" >> $LOGFILE
        else
            echo "$(date): Failed to mount /dev/drbd0 to /data" >> $LOGFILE
            exit 1https://chatgpt.com/c/6bcf1581-5ad4-4fef-be2a-0e81a9598aa3
        fi
    fi

    echo "$(date): DRBD promotion and mount script completed" >> $LOGFILE
    ```
- make it executable: `chmod +x /root/scripts/mount.sh `
- create service `nano /etc/systemd/system/data.mount`:
    ```
    [Unit]
    Description=Promote DRBD Resource to Primary
    After=drbd@r0.service

    [Service]
    Type=oneshot
    ExecStart=/root/scripts/mount.sh
    RemainAfterExit=true

    [Install]
    WantedBy=multi-user.target
    ```
- enable services:
   - `systemctl enable drbd@r0.target`
   - `systemctl enable drbd-wait-promotable@r0.service`
   - `systemctl enable data.mount`
- `reboot`

Verify it works:

- show the current transmission state: `cat /proc/drbd`
- show the status of r0: `drbdadm status r0`


# k3s installation

on master:
```shell
mkdir /data/k3s
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--data-dir /data/k3s" sh -
cat /data/k3s/server/node-token # this prints the token for adding a node
```

on worker node:
```shell
curl -sfL https://get.k3s.io | K3S_URL=https://k3s-master:6443 K3S_TOKEN=<TOKEN> sh -
```
