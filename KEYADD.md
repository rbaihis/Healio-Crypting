##  Create a LUKS Key File:
```bash
sudo mkdir -p /etc/luks-keys
sudo dd if=/dev/urandom of=/etc/luks-keys/luks-key.bin bs=512 count=1
sudo chmod 400Or600 /etc/luks-keys/luks-key.bin
```
Commands Explained:
>> 1. **dd** -> dd utility to create a file containing random data (dd can duplicate data across files, devices, partitions and volumes).
>> 2. **if=/dev/urandom** -> This specifies the input file for dd.
>> 3. **of=/SomePath** -> This specifies the output file for dd.
>> 4. **bs=512** -> This sets the block size to 512 bytes.  dd reads and writes data in blocks.
>> 5. **count=1** -> This tells dd to copy only one block.  Since the block size is 512 bytes, this command will create a 512-byte file.  This is a common size for a LUKS key.

##  IAdd the Key to Your LUKS Volume
> Identify encrypted partition 
```bash
sudo blkid | grep LUKS
```
> Add the Key to Your LUKS Headers of The Crypted Volume:
```bash
sudo cryptsetup luksAddKey /dev/TheDiskOrPartition /etc/luks-keys/luks-key.bin
```
>>Ps:  You’ll be prompted to enter your existing LUKS passphrase.
>> You re done


---
##  Automate Key on boot (case not a critical volume for boot)
> Crypttab file
``` bash
# <target name> <source device>         <key file>      <options>
#volumeNameToGenerateWhenDecrypt(Mountable in fstab)    /dev/theDevice(or UUID=...)  /pathToKey  luks
#data_crypted UUID=c71c5757-dcbb-47ff-80c4-5d3936eee20a /etc/luks-keys/luks-key.bin     luks
data_crypted UUID=c71c5757-dcbb-47ff-80c4-5d3936eee20a  none    luks
```
---
> fstab file
```bash
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/ubuntu-vg/ubuntu-lv during curtin installation
/dev/disk/by-id/dm-uuid-LVM-UNiHN1kv7e2eydyHOl5slQb6xnVNI6S1m300fz1WM3V9KMWmuPcdFRQDUXHcoeFt / ext4 defaults 0 1
# /boot was on /dev/sda2 during curtin installation
/dev/disk/by-uuid/01fb01c6-3215-48e8-b5ac-06b99c1917ab /boot ext4 defaults 0 1
/swap.img       none    swap    sw      0       0
/dev/mapper/data_crypted        /mnt/my_encrypted_data  ext4  defaults 0 2

```

