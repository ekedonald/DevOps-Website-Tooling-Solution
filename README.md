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