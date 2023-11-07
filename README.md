# DevOps Tooling Website Solution
 In this project, I will implement a solution that consists of the following components:
 1. **Infrastructure**: AWS
 2. **Web Server Linux**: Red Hat Enterprise Linux 9
 3. **Database Server**: Red Hat Enterprise Linux 9 + MySQL
 4. **Storage Server**: Red Hat Enterprise Linux 9 + NFS Server
 5. **Programming Language**: PHP
 6. **Code Repository**: GitHub

On the diagram below, you can see a common pattern where several stateless Web Servers share a common database and also access the same files using [Network File System (NFS)](https://en.wikipedia.org/wiki/Network_File_System) as a shared file storage. Even though the NFS server might be located on a completely seperate hardware (_i.e. for Web Servers it will look like a local file system from where they can serve the same files_).

It is important to know what storage solution is suitable for certain use cases. To determine this, you need to answer the following questions: **what data will be stored, in what format, how this data will be accessed, by whom, from where and how frequently**. 
 
Based on this you will be able to choose the right storage system for your solution.

## How To Implement a DevOps Tooling Website Solution
The following steps are taken to implement a DevOps Tooling Website Solution:

### Step 1: Provision an NFS Server EC2 Instance

Use the following parameters when configuring the EC2 Instance:

1. Name of Instance: Web Server
2. AMI: Red Hat Enterprise Linux 9 (HVM), SSD Volume Type
3. New Key Pair Name: web11
4. Key Pair Type: RSA
5. Private Key File Format: .pem
6. New Security Group: NFS Server SG
7. Inbound Rules: Allow Traffic From Anywhere On Port 22

### Step 2: Set additional Inbound Rules to allow connections from the NFS Clients (i.e. The 3 Web Servers) on the NFS Port.

* On the Instance Summary for the NFS Server shown above, click on the Subnet ID.

* Copy the Subnet IPv4 CIDR address.

* Add rules that allow connections from the Subnet CIDR on TCP Port 2049, TCP Port Port 111, UDP Port 2049 and UDP Port 111.

### Step 3: Create and Attach 3 Elastic Block Store Volumes to the NFS Server EC2 Instance

* On the Instances tab, notice the **Availabilty Zone (i.e. us-east-1d)** of the NFS Server Instance. This will be used to configure the 3 EBS Volumes.

* On the EC2 dashboard, click on the **Volumes** on the Elastic Block Store tab.

* Click on the Create Volume button.

* Give the EBS Volume the following parameters and click on the **create volume** button:

1. Size (GiB): 10
2. Availability Zone: us-east-1d (_Note that the Availability Zone you select must match the Availability zone of the NFS Server Instance_)

* Repeat the steps above to create two more EBS Volumes.

_You will see the 3 EBS Volumes you created have an Available Volume state_

* Click on one of the volumes then click on the **Actions** button, you will see a drop-down and click on the **Attach volume** option.

* Select the NFS Server Instance and click on the **Attach volume** button.

* Repeat these steps for the other 2 volumes and you will see that the volumes have been attached to the NFS Server Instance as shown below:

### Step 4: Implement LVM Storage Management on the NFS Server

* Open terminal on your computer.

* Go to the Downloads directory (_i.e. `.pem` key pair is stored here_) using the command shown below:

```sh
cd Downloads
```

* Run the following command to give read permissions to the `.pem` key pair file.

```sh
chmod 400 <private-key-pair-name>.pem
```

* SSH into the NFS Server Instance using the command shown below:

```sh
ssh -i <private-key-name>.pem ubuntu@<Public-IP-address>
```

* Use the `lsblk` command to inspect the block devices attached to the server.

_Notice the names of the new created devices._

* Use `gdisk` utility to create a single partiton on **/dev/xvdf** disk.

_Note that all the devices in Linux reside in the **/dev** directory_

```sh
sudo gdisk /dev/xvdf
```

* Type `n` to create a new partiton and fill in the data shown below into the parameters:

1. Partition number (1-128, default 1): 1
2. First sector (34-20971486, default = 2048) or {+-}size{KMGTP}: 2048
3. Last sector (2048-20971486, default = 20971486) or {+-}size{KMGTP}: 20971486
4. Current type is 8300 (Linux filesystem) Hex code or GUID (l to show codes, Enter = 8300): 8300

* Type `p` to print the partition table of the **/dev/xvdf** device.

* Type `w` to write the table to disk and type `y` to exit.

* Repeat the `gdisk` utility partitioning steps for **/dev/xvdg** and **/dev/xvdh** disks.

* Use the `lsblk` command to view the newly configured partiton on each of the 3 disks.

* Install `lvm2` package using the command shown below:

```sh
sudo yum install lvm2 -y
```

* Run the following command to check for available partitons:

```sh
sudo lvmdiskscan
```

* Use `pvcreate` utility to mark each of the 3 disks as physical volumes (PVs) to be used  by LVM.

```sh
sudo pvcreate /dev/xvdf1
sudo pvcreate /dev/xvdg1
sudo pvcreate /dev/xvdh1
```

* Verify that your physical volumes (PVs) have been created successfully by running `sudo pvs`

* Use `vgcreate` utility to add 3 physical volumes (PVs) to a volume group (VG). Name the volume group **webdata-vg**.

```sh
sudo vgcreate webdata-vg /dev/xvdf1 /dev/xvdg1 /dev/xvdh1
```

* Verify that your volume group (VG) has been created successfully by running `sudo vgs`

* Use the `lvcreate` utility to create 3 logical volumes: apps-lv (_use a third of the PV size_), logs-lv (_another third of the PV size_) and opt-lv (_the remaining third of the PV size_).

```sh
sudo lvcreate -n apps-lv -L 9.5G webdata-vg
sudo lvcreate -n logs-lv -L 9.5G webdata-vg
sudo lvcreate -n opt-lv -L 9.5G webdata-vg
```

* Verify that your logical volume (LV) has been created successfully by running `sudo lvs`

* Verify the entire setup by running the following commands:

```sh
sudo vgdisplay -v #view complete setup - VG, PV, and LV
```

* Use `mkfs.xfs` to format the logical volumes (LV) with **xfs** file system.

```sh
sudo mkfs -t xfs /dev/webdata-vg/apps-lv
```

```sh
sudo mkfs -t xfs /dev/webdata-vg/logs-lv
```

```sh
sudo mkfs -t xfs /dev/webdata-vg/opt-lv
```

* Create **/mnt/apps** directory to be used by the Web Servers.

```sh
sudo mkdir -p /mnt/apps
```

* Create **/mnt/logs** directory to be used by Web Servers log.

```sh
sudo mkdir -p /mnt/logs
```

* Create **/mnt/opt** directory to be used by Jenkins Server.

```sh
sudo mkdir -p /mnt/opt
```

* Mount **/mnt/apps** on **apps-lv** logical volume.

```sh
sudo mount /dev/webdata-vg/apps-lv /mnt/apps
```

* Use `rsync` utility to backup all the files in the log directory **/var/log** into **/mnt/logs** (*This is required before mounting the file system*).

```sh
sudo rsync -av /var/log/. /mnt/logs
```

* Mount **/var/log** on **logs-lv** logical volume. (_Note that all the existing data on /var/log will be deleted_).

```sh
sudo mount /dev/webdata-vg/logs-lv /var/log
```

* Restore log files back into **/var/log** directory.

```sh
sudo rsync -av /mnt/logs/ /var/log
```

* Update `/etc/fstab` file so that the mount configuration will persist after restarting the server. The UUID of the device will be used to update the `/etc/fstab` file. Run the command shown below to get the UUID of the **apps-lv**, **logs-lv** and **opt-lv** logical volumes:

```sh
sudo blkid
```

* Update `/etc/fstab` in this format using your own UUID and remember to remove the leading and ending quotes.

```sh
sudo vi /etc/fstab
```

* Test the configuration using the command shown below:

```sh
sudo mount -a
```

* Reload the daemon using the command shown below:

```sh
sudo systemctl daemon-reload
```

### Step 4: Install and configure the NFS Server.

