# Luks Container Volume Steps
## Login As Root (To avoid missing sudo)
```
# 99% of steps require root privelege
sudo su
```

#### Create a big file to act as our Container
> bs (bufferSize) * count = size of file

> use urandom instead of zero, for better block allocation

>> Note!:
>>> make sure to `never delete or override` this file created ever.

>>> place it in a place that u never need to use it.
```bash
sudo dd if=/dev/urandom of=/container_critical_fs bs=1M count=10240 status=progress
```


#### Format/Create file as lucks volume (fake file style volume , AKA container volume):
> `This action will make the file as a volume, and format it`.
>> Use type `luks2`

>> U will be prompt to enter passphrase make sure to enter Very strong passphrase + (`backup` it)
```bash

# create luks volume
# Type capital Letter YES
# Enter Very Very Strong Passphrase
sudo cryptsetup luksFormat --type luks2 /container_critical_fs

```

#### Create key-file for automatic unlock in cryptab
> **Note:** `up to 9 Keys the luks header can keep track of`
>> Make sure to use `urandom` for security.
>>> Make sure its `0400` 
>>>> This example place the key `in obvious location, try to place it in a more hidden place 
```bash
mkdir -p /etc/luks-keys 
sudo dd if=/dev/urandom of=/etc/luks-keys/critical_volume.key bs=512 count=8
sudo chmod 0400 /etc/luks-keys/critical_volume.key
```

**Add the generated key file To the luck volume is current header**
```bash
# u will be prompt to enter your passphrase to accept the key
sudo cryptsetup luksAddKey /container_critical_fs /etc/luks-keys/critical_volume.key
```


#### Backup header before first use 
> Crytical operation
> **Make sure** to store the `backup Header, key and passphrase` in a safe place for `disaster recovery in case of corrupted header`
>> **Destination of detached backup Header**:
>>> `/boot/luks_header_critical.img`
```bash
sudo cryptsetup luksHeaderBackup /container_critical_fs --header-backup-file /boot/luks_header_critical.img

#Secure header permission
sudo chmod 600 /boot/luks_header_critical.img 
```

#### Open The crypted Volume (using the detached header + key)
> `crypted_volume` the output of the command a device mapper called : `/dev/mapper/crypted_volume`
>> open the luks container to act as a volume mapped .
> Note: The header here is not necessary we only use it to test  (should work fine without it `as long the current header knows about the used decrypt-key in its metadata`)

```bash
# Pre check
ls /dev/mapper

# Create the volume mapper by decrypt/open the lucks volume
sudo cryptsetup open --header /boot/luks_header_critical.img --key-file=/etc/luks-keys/critical_volume.key /container_critical_fs crypted_volume

# Post check
ls /dev/mapper
```


#### Create a fileSystem on the mapped volume  
> **Note:** this operation should be executed `one time only` (will delete data if redone later)
>> `mkfs.ext4` file system of type ext4
```bash
sudo mkfs.ext4  /dev/mapper/crypted_volume  
```


#### Force fsck every 30 mounts 
>30 is good value lower if necessary.
>> Good to do for filesystem check/repair since we re not dealing with a real partion or disk
```bash
# Force fsck every 30 mounts
tune2fs -c 30 -i 180d /dev/mapper/crypted_volume  
```

#### Save the UUID, we will need it for automated mount later
> Make sure to `not skip` this operation
```bash

blkid /dev/mapper/crypted_volume # copy uuid 
# save the UUID output in a file (example)
#  UUID="6e853d45-c269-4d57-9ea1-ac3b60507b90"
```

#### Mount The ext4.formated volume on a mount point
Create a repo to act as mounted point in 
> make sure `repo is new`
```bash
mkdir -p /mnt/targa_crypted
```

**Manual Test** Your volume By Manual Mount
> Device mapper should be already existing with name `/dev/mapper/crypted_volume`
```bash
mount /dev/mapper/crypted_volume /mnt/targa_crypted
```

Create a file for testing:
```bash
# Pre Check
findmnt /mnt/targa_crypted
ls -l /mnt/targa_crypted

# create a test file, do not delete it later we will use to check its existing in reboot
echo "hello targa testing my crypted volume" >> /mnt/targa_crypted/hello.txt

# Post check
ls -l /mnt/targa_crypted
du -sh /mnt/targa_crypted
```

`Unmount` then `Mount` again to check:
```bash
umount /mnt/targa_crypted

# should not show the hello.txt
ls -l /mnt/targa_crypted 
du -sh /mnt/targa_crypted

# mount again to see the hello.txt exist
mount /dev/mapper/crypted_volume /mnt/targa_crypted

ls /mnt/targa_crypted

# unmount test is finished (we will automate in boot now)
umount /mnt/targa_crypted
``` 

#### (/etc/crypttab) Automate Decrypt Operation at boot time
> This operation will automate the manual command on boot time `sudo cryptsetup open --header /boot/luks_header_critical.img --key-file=/etc/luks-keys/critical_volume.key /container_critical_fs crypted_volume`
> Option used:
>> `header=/boot/....` use detatched header saved, (can be ommited).

>> `nofail` if operation failed do not stuck in boot and ignore the operation.
```bash

# (best practice)backup original crypttab 
mv /etc/crypttab /etc/crypttab.bak

cp /etc/crypttab.bak /etc/crypttab
chown root:root /etc/crypttab
chmod 0644 /etc/crypttab

# add line to /etc/fstab
echo "crypted_volume /container_critical_fs /etc/luks-keys/critical_volume.key luks,header=/boot/luks_header_critical.img,nofail" >> /etc/crypttab
```


#### (/etc/fstab) Automate Mount at boot time 
> !! The mount Point repo should exist

> This operation will automate the manual command on boot time `mount /dev/mapper/crypted_volume /mnt/targa_crypted`

> Options Used Worth Noting:
>> `nofail` if operation failed do not stuck in boot and ignore the operation.

>> `defaults` A set of common default options: rw, relatime, errors=remount-ro, exec, auto, nouser, and async.

>> `noatime` Disables updating access time on reads — improves performance (useful for SSDs or logging). 

>> `2` For non critical partitions (checked after root) since its for data and user application in our case.

```bash
#UUID="6e853d45-c269-4d57-9ea1-ac3b60507b90"

# (best practice)backup original crypttab 
mv /etc/fstab /etc/fstab.bak

cp /etc/fstab.bak /etc/fstab
chown root:root /etc/fstab
chmod 0644 /etc/fstab

# U need the UUId that u where asked to save it earlier using `blkid /dev/mapper/crypted_volume`
echo "UUID=xxx-xxx-xx /mnt/targa_crypted ext4 defaults,noatime,data=ordered,nofail 0 2" >> /etc/fstab
```

#### Reboot and test to see if your luks volume is decrypted and Mounted
> U should see the `hello.txt` if you didn't delete it.
```bash
# to test the auto decrypt and mount
reboot

# check the mount is done and file exist
# if mount exist =>(decrypt+mount == success)
findmnt /mnt/targa_crypted
ls /mnt/targa_crypted

# All good we re done with luks
```


#### Test Emergency Recovery 
> Scenario : 
>> Current `header is corrupted`.

>> U posess `older backup header` + `key or passphrase` compatible with the header 
```bash
# access as root after reboot
sudo su 

# we unmount and close luks to assume we have a crypted luks that we can't access
# luks close operation make the data crypted unaccessible unless u have the key or passphrase that is compatible with current header , or an old backup header with its corresponding key or passphrase
umount /mnt/targa_crypted
sudo cryptsetup luksClose crypted_volume
# check no device mapper with name /dev/crypted_volume exist
ls /dev/mapper

# We will pretend using old header with compatiblepassphrase to open the luks volume
sudo cryptsetup open --header /boot/luks_header_critical.img  /container_critical_fs crypted_volume

# verify if the volume is mapped
ls /dev/mapper

# done thats all
```

---
# Part 2 : Mapping Postgres Data to The mount Point

## Method 1: Postgres Everything Reside in Luks 

#### Assuming Postgres Intalled already  
```bash
# reboot system to start working clean, since we didn't bother with the mount phase in the previous test and we didn't remount it 
reboot
# back as root
sudo su
# install it if not existing to simulate it
sudo apt update 
#sudo aot upgrade # pointless since testing
sudo apt install postgresql -y
```


#### Importing All Postgres in the new Destination

```bash

# stop all postgres service
sudo systemctl stop postgresql
sudo systemctl stop postgresql@14-main.service
sudo systemctl status postgresql.service
sudo systemctl status postgresql@14-main.service

# check if mount is propper (double checking) [unnecessary sould work out of the box]
findmnt /mnt/targa_crypted

# Create potgres repo inside the luks volume
sudo mkdir -p /mnt/targa_crypted/postgresql
# use `rsync` only copy all postgres  
# can redo rsync as many time safe and idempotent
sudo rsync -aAXv --delete --progress /var/lib/postgresql/ /mnt/targa_crypted/postgresql/
#verify if same repo access rights as the original
#verify if same repo access rights as the original
ls -ld /mnt/targa_crypted/postgresql

#if ownership is not postgres change owner
sudo chown -R postgres:postgres /mnt/targa_crypted/postgresql

# fast check same size
du -sh /var/lib/postgresql 
du -sh /mnt/targa_crypted/postgresql

# after successful rsync
# backup postgres original folder by rename
# use mv , do not use cp to maintain permission and ownership in the backup
sudo mv /var/lib/postgresql /var/lib/postgresql.bak   
```


#### (Auto-Binding)  /mnt/../postgresql to /var/../postgresql
> This operation will auto bound in restart
> we will test all later when mentioned
```bash
# this for auto bind on reboot
echo "/mnt/targa_crypted/postgresql /var/lib/postgresql none bind 0 0"  >> /etc/fstab
```


####  Modify Postgres Service To wait for Mount or fail if not exist
> this setup is to (add extra options) to force postgres to use the crypted volume [safety measures]

> Option used:

>> `After=mnt-targa_crypted.mount`:
>>> this will tell postgres to wait until the volume is mounted.

>> `Requires=mnt-targa_crypted.mount`:
>>> this will tell postgres that it requires the mount.

>> `ExecStartPre=/bin/mountpoint /var/lib/postgresql`
>>> This test if the specifyed folder is mounted. if it return '1->failure' postgres will not start.

```bash
[Unit]
# will fail to start if luks volume is not decrypted
After=dev-mapper-crypted_volume.device mnt-targa_crypted.mount
# will try the mount (even if its unmounted - safe and good since lucks is already decrypted)
Requires=dev-mapper-crypted_volume.device mnt-targa_crypted.mount

[Service]
# Verifies that the LUKS volume is decrypted and available as a block device.
ExecStartPre=/bin/sh -c '[ -b /dev/mapper/crypted_volume ] || { echo "ERROR: LUKS not decrypted"; exit 1; }'


# 2. Check crypted volume is mounted
ExecStartPre=/bin/sh -c 'mountpoint -q /mnt/targa_crypted || { echo "ERROR: /mnt/targa_crypted not mounted"; exit 1; }'

# Combined check in one simple command
# This line ensures all of the following:
# The LUKS volume is available
# /mnt/targa_crypted is mounted
# /var/lib/postgresql is also mounted
# The source of the /var/lib/postgresql mount is somewhere under /dev/mapper/crypted_volume and includes postgresql (to match something like a bind mount from /mnt/targa_crypted/postgresql).
ExecStartPre=/bin/sh -c 'if ! { [ -b /dev/mapper/crypted_volume ] && \
mountpoint -q /mnt/targa_crypted && \
mountpoint -q /var/lib/postgresql && \
findmnt -n -o SOURCE /var/lib/postgresql | grep -q "/dev/mapper/crypted_volume.*postgresql";>
echo "ERROR: Mount verification failed"; exit 1; fi'

TimeoutStartSec=90
```

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
```

#### Test that postgres will run correctly 
```bash
# reboot the system so the automated --bound take effect
reboot 

# Check if mount occured (it should be there)
findmnt /mnt/targa_crypted
# Check if mount occured (it should be there)
findmnt /var/lib/postgresql

# if:
    # Decrypt -> successful
    # mount /dev/mapper/crypted_volume /mnt/targa_crypt) -> successful
    # mount --bind /mnt/targa_crypted/postgresql /var/lib/postgresql -> successful
# ==> postgres will be started and running
systemctl status  postgresql@14-main.service
```


#### Verify Test That postgres will fail if luks is uncrypted bind exist (service override works as desired)
```bash
# stop both postgres services:
sudo systemctl stop  postgresql.service
sudo systemctl stop  postgresql@14-main.service

# unbound the /var/lib/postgres mount
sudo umount /var/lib/postgresql
sudo ls /var/lib/postgresql #==> empty
sudo findmnt /var/lib/postgresql # ==> no output
sudo umount /mnt/targa_crypted
sudo ls /mnt/targa_crypted #==> empty
sudo findmnt /mnt/targa_crypted # ==> no output
sudo cryptsetup luksClose crypted_volume 
ls /dev/mapper

# restart postgresql
sudo systemctl start  postgresql.service
# first service is often enough this just to force the second second and test it
sudo systemctl start  postgresql@14-main.service

# Check Status should fail:
sudo systemctl status  postgresql@14-main.service

# to restore decrypt and mount config without reboot after test use:
sudo cryptsetup open --header /boot/luks_header_critical.img  /container_critical_fs crypted_volume
sudo mount /dev/mapper/crypted_volume /mnt/targa_crypted
sudo mount --bind /mnt/targa_crypted/postgresql /var/lib/postgresql

```


#### Test that Service will mount unmounted volumes assuming they re unmounted
> (same scenario without closing luks)
```bash
# stop both postgres services:
sudo systemctl stop  postgresql.service
sudo systemctl stop  postgresql@14-main.service

# unbound the /var/lib/postgres mount
sudo umount /var/lib/postgresql
sudo ls /var/lib/postgresql #==> empty
sudo findmnt /var/lib/postgresql # ==> no output
sudo umount /mnt/targa_crypted
sudo ls /mnt/targa_crypted #==> empty
sudo findmnt /mnt/targa_crypted # ==> no output


# restart postgresql
sudo systemctl start  postgresql.service
# first service is often enough this just to force the second second and test it
sudo systemctl start  postgresql@14-main.service

# Check Status should start and mount happenes implicitly
sudo systemctl status  postgresql@14-main.service

# check mount exist
sudo findmnt /var/lib/postgresql # ==> no output
sudo findmn /mnt/targa_crypted
```

--- 

## Method2: (Just point the data_dir only to the desired mounted folder )
> good for odoo, also postgres can be configured this way
```bash
# To Do Later
# -1- sudo findmnt /mnt/targa_crypted ==> assure its mounted from /dev/mapper/theCryptedVolumeName

# 0- create a repo (eg, mkdir -p /mnt/targa_crypted/postgresql/14/data_dir)

# 1- Stop postgres services (stop both 2 just in case)

# 3- rsync your datadir to the new location (keep ownership and permission)
    # (eg, sudo rsync -aAXv --delete --progress /var/lib/postgresql/14/main/ /mnt/targa_crypted/postgresql/14/data_dir)

# 4- in postgresql configuration:
    # /etc/postgresql/14/main/postgresql.conf
    # data_directory = '/mnt/targa_crypted/postgresql/14/data_dir' 

# 5- edit your postgresql@version service like method one to safe start only when all mounted

# 6- restart service should work fine. (test it)

# 5- reboot and check systemd. (test it) 
```
