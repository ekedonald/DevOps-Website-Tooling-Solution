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

### Step 4: Install and configure the NFS Server.

