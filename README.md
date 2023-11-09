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

* Add rules that allow connections from the Subnet CIDR (_i.e. **172.31.16.0/20**_) on TCP Port 2049, TCP Port Port 111, UDP Port 2049 and UDP Port 111.

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
ssh -i <private-key-name>.pem ec2-user@<Public-IP-address>
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

* Mount **/mnt/opt** on **opt-lv** logical volume.

```sh
sudo mount /dev/webdata-vg/opt-lv /mnt/opt
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

* Verify your setup by running `df -h`

### Step 5: Install and configure the NFS Server.

* Update the list of packages in the package manager.

```sh
sudo yum -y update
```

* Install the NFS Server package.

```sh
sudo yum install nfs-utils -y
```

* Start the NFS Server service.

```sh
sudo systemctl start nfs-server.service
```

* Enable the NFS Server service.

```sh
sudo systemctl enable nfs-server.service
```

* Check the status of the NFS Server service.

```sh
sudo systemctl status nfs-server.service
```

* Allow read, write and execute permissions for the Web Servers on the NFS Server.

```sh
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt
```

* Restart the NFS Server service.

```sh
sudo systemctl restart nfs-server.service
```

* Configure access to NFS for clients (_i.e. Web Servers_) within the same subnet (**Subnet CIDR: 172.31.16.0/20**).

```sh
sudo vi /etc/exports
```

```sh
/mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/opt <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
```

```sh
sudo exportfs -arv
```

* Check which port is used by NFS. **Note that Inbound Rules have already been set to allow connections from the client (i.e Web Servers) on the NFS Port (i.e TCP and UDP Ports: 2049 and 111)**.

```sh
rpcinfo -p | grep nfs
```

### Step 6: Provision a Database Server EC2 Instance

1. Name of Instance: Database Server
2. AMI: Red Hat Enterprise Linux 9 (HVM), SSD Volume Type
3. Key Pair Name: web11
4. New Security Group: Database Server SG
Inbound Rules: Allow Traffic From Anywhere On Port 22 and Traffic from the Subnet CIDR on Port 3306 (i.e. MySQL).

_Instance Summary for Database Server_

### Step 7: Configure the Backend Database as part of the 3-Tier Architecture

* Open another terminal on your computer.

* Go to the Downloads directory (_i.e. `.pem` key pair is stored here_) using the command shown below:

```sh
cd Downloads
```

* SSH into the Database Server Instance using the command shown below:

```sh
ssh -i <private-key-name>.pem ec2-user@<Public-IP-address>
```

* Update the list of packages in the package manager.

```sh
sudo yum update -y
```

* Install MySQL server.

```sh
sudo yum install mysql-server -y
```

* Start the MySQL service.

```sh
sudo systemctl start mysqld
```

* Enable the MySQL service.

```sh
sudo systemctl enable mysqld
```

* Check if MySQL service is up and running.

```sh
sudo systemctl status mysqld
```

* Log into the MySQL console application.

```sh
sudo mysql
```

* Create a database called `tooling`.

```sh
CREATE DATABASE tooling;
```

* Create a new user.

```sh
CREATE USER 'myuser'@'<Subnet_CIDR' IDENTIFIED BY 'password';
```

* Grant all privileges on the `tooling`database to the user created.

```sh
GRANT ALL ON tooling.* TO 'myuser'*'<Subnet_CIDR';
```

* Run the following command to apply and make changes effective.

```sh
FLUSH PRIVILEGES;
```

* Display all the databases.

```sh
SHOW DATABASES;
```

* Exit the MySQL console.

### Step 8: Provision 3 Web Servers EC2 Instances

1. Name of Instance: Web Server 1 (_change the names of the other two Instances to Web Server 2 and Web Server 3_)
2. AMI: Red Hat Enterprise Linux 9 (HVM), SSD Volume Type
3. Key Pair Name: web11
4. New Security Group: Web Server SG
Inbound Rules: Allow Traffic From Anywhere On Port 22 and Port 80

### Step 9: Configure the Web Servers

* Open another terminal on your computer.

* Go to the Downloads directory (_i.e. `.pem` key pair is stored here_) using the command shown below:

```sh
cd Downloads
```

* SSH into the Database Server Instance using the command shown below:

```sh
ssh -i <private-key-name>.pem ec2-user@<Public-IP-address>
```

* Install NFS Client

```sh
sudo yum install nfs-utils nfs4-acl-tools -y
```

* Mount `/var/www` and target the NFS server's export for apps.

```sh
sudo mkdir /var/www
```

```sh
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private_IP-Address>:/mnt/apps /var/www
```

* Mount apache's log folder to the NFS server's export for logs.

```sh
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private_IP-Address>:/mnt/logs /var/log
```

* Verify that NFS was mounted successfully by running `df -h`

* Make sure that the changes will persist on the Web Server after reboot by updating the `/etc/fstab`

```sh
sudo vi /etc/fstab
```

* Add the following line then save and exit the file:

```sh
<NFS-Server-Private-IP-Address>:/mnt/apps /var/www nfs defaults 0 0
<NFS-Server-Private-IP-Address>:/mnt/logs /var/log nfs defaults 0 0
```

* Test the configuration using the command shown below:

```sh
sudo mount -a
```

* Reload the daemon using the command shown below:

```sh
sudo systemctl daemon-reload
```

* Install [Remi's repository](http://www.servermom.org/how-to-enable-remi-repo-on-centos-7-6-and-5/2790/), Apache and PHP

```sh
sudo yum install httpd -y

sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm -y

sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm -y

sudo dnf module reset php

sudo dnf module enable php:remi-7.4

sudo dnf install php php-opcache php-gd php-curl php-mysqlnd -y

sudo systemctl start php-fpm

sudo systemctl enable php-fpm

sudo setsebool -P httpd_execmem 1

sudo systemctl start httpd

sudo systemctl enable httpd
```

* Open two terminals and SSH into Web Server 2 and Web Server 3 EC2 Instances and run the following command to configure the two Web Servers:

```sh
sudo vi install.sh
```

* Paste the codebase below then save and exit the file.

```sh
#!/bin/bash

# input the Private IPv4 of your NFS Server
nfs_server_private_ip=172.31.26.52

sudo yum install nfs-utils nfs4-acl-tools -y

sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid $nfs_server_private_ip:/mnt/apps /var/www
sudo mount -t nfs -o rw,nosuid $nfs_server_private_ip:/mnt/logs /var/log
sudo mount -a
sudo systemctl daemon-reload

sudo chmod 777 /etc/fstab
sudo echo "$nfs_server_private_ip:/mnt/apps /var/www nfs defaults 0 0" >> /etc/fstab
sudo echo "$nfs_server_private_ip:/mnt/logs /var/log/httpd nfs defaults 0 0" >> /etc/fstab
sudo chmod 644 /etc/fstab

sudo yum install httpd -y
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm -y
sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm -y
sudo dnf module reset php
sudo dnf module enable php:remi-7.4
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd -y
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo setsebool -P httpd_execmem 1
sudo systemctl start httpd
sudo systemctl enable httpd
```

* Run the following command to run the `install.sh` script.

```sh
bash install.sh
```

* Verify that apache files and directories are available on Web Servers in `/var/www` and also on the NFS server in `/mnt/apps`. If you see the same files, it means NFS is mounted correctly. Verification can be done by taking the following steps:

1. On the Web Server 1 Terminal, go to the `/var/www/` directory and create a `test.txt` file in the `/var/www` directory

2. On the NFS Server Terminal, go to the `/mnt/apps` directory and run the `ll` command to view list the files in the directory. You will see that the file `test.txt` file is present.

### Step 10: Fork the tooling source code from [Darey.io GitHub account](https://github.com/darey-io/tooling)

* Check if git in installed on the Web Server using the following command:

```sh
which git
```

* Install the git package.

```sh
sudo yum install git -y
```

* Go to the [Darey.io Tooling Repository](https://github.com/darey-io/tooling) and copy the highlighted link shown below:

* Clone the repository.

```sh
git clone https://github.com/darey-io/tooling.git
```

* Deploy the tooling website's code to the Web Server. Ensure that the **html** folder from the repository is deployed to `/var/www/html`

```sh
cd tooling && ll
```

```sh
sudo cp -r html/. /var/www/html/
```

* Disable SELinux.

```sh
sudo setenforce 0
```

* To make this change permanent, open the following configuration file `/etc/sysconfig/selinux` and set `SELINUX=disabled`

```sh
sudo vi /etc/sysconfig/selinux
```

* Update the website's configuration to connect to the Database Server by running the following command:

```sh
sudo vi /var/www/html/functions.php
```

* Install MySQL client.

```sh
sudo yum install mysql -y
```

* Apply `tooling-db.sql` script to your database using this commands shown below:

```sh
cd tooling
```

```sh
mysql -h <database-private-ip> -u <db-username> -p tooling < tooling-db.sql
```

### Step 11: Create a new admin user on your Database Server

* Connect to the Database Server Instance.

* Log into the console application.

```sh
sydo mysql
```

* Display all the databases.

```sh
SHOW DATABASES;
```

* Select the `tooling` database you want to work on.

```sh
USE tooling;
```

* Display the tables in the `tooling` database.

```sh
SHOW TABLES;
```

* Display all the contents of the `users` table.

```sh
SELECT * FROM users;
```

* Input data of a new user into the table.

```sh
INSERT INTO users (id, username, password, email, user_type, status)
-> VALUES (2, 'donald', '5f4dcc3b5aa765d61d8327deb882cf99', 'user@mail.com', 'admin', '1')
```

* Exit the console application.

### Step 12: Start and Enable Apache on Web Server.

* Connect to the Web Server 1 Instance.

* Run the following command to test if you can connect to the tooling website:

```sh
curl localhost
```

* Start the apache service.

```sh
sudo systemctl start httpd
```

* Enable the apache service.

```sh
sudo systemctl enable httpd
```
