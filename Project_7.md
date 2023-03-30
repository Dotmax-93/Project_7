## IMPLEMENTATION OF PROJECT 7

# Create an instance that will serve as NFS server(Redhat)
on the created instance, attach 2 volumes each containing 10gib and attach it to the AZ of the created instance, add TCP 111, UDP 111, UDP 2049 to the security groups

`lsblk` [^5]![Created volumes]

[^5]: This is done to view how the volumes are attached on the server

`ls /dev` [^9]

[^9]:This command shows the newly created volumes

`df -h` [^13]

# create a partition for each volume created

[^13]:This command shows how the volumes are mounted and the number of spaces

`sudo gdisk /dev/xvdb`    `sudo gdisk /dev/xvdc` [^19]![Created partitions]

[^19]:This command is used to create a single partition on the created volumes,so we do the same for each volume created

# Install logical volumes for each partition

`sudo yum install lvm2` 

`sudo lvmdiskscan ` [^25]![Installed lvm]

[^25]:This command installs the logical volume to check for availsble partition

# Mark each disk as physical lovems to be used by the logical volumes

`sudo pvcreate /dev/xvdb1 `

`sudo pvcreate /dev/xvdc1 `


[^33]:This is created for each attahced volume to mark them as physical volume to be used by lvm

`sudo pvs` [^41]![created PVs]

[^41]: This is done to verify that our pvs is working successfully

# Add all PVs  to a volume group 

`sudo vgcreate nfs-vg /dev/xvdb1 /dev/xvdc1` [^47]

 [^47]:We use this comman to add the PV to a volume group named webdata-vg

 `sudo vgs` [^51]![created vgs]

 [^51]:This command verifes that our volume group has been created successfully

 # create 2 logical volumes, one for the website and the other for the logs

 `sudo lvcreate -n lv-apps -L 5G nfs-vg` [^57]

 `sudo lvcreate -n lv-logs -L 5G nfs-vg`

 `sudo lvcreate -n lv-opt -L 5G nfs-vg`

 [^57]:This command creates 2 logical volumes namely apps and logs to store data for website and logs

 `sudo lvs` [^63]![alt text](./Images/6.png)
 
 [^63]: This command confirms that the lv is up and running

 `sudo vgdisplay -v #view complete setup - VG, PV, and LV` [^67]![alt text](./Images/7.png)

`sudo lsblk ` ![alt text](./Images/9.png)

[^67]: This command confirms all set up on our webserver

`sudo mkdir /mnt/apps /mntlogs /mnt/opt`

# Format the logical volumes using xfs

`sudo mkfs.xfs /dev/nfs-vg/lv-apps `

`sudo mkfs.xfs /dev/nfs-vg/lv-logs `

`sudo mkfs.xfs /dev/nfs-vg/lv-opt `

# create mount points on /mnt directory for the logical volumes
`sudo mount /dev/nfs-vg/lv-apps /mnt/apps `

`sudo mount /dev/nfs-vg/lv-logs /mnt/logs `

`sudo mount /dev/nfs-vg/lv-opt /mnt/opt `

# Copy the UUID of apps,logs and opt to /etc/fstab

`sudo blkid /dev/nfs-vg/*` [^105]

`sudo vi /etc/fstab` [^96]

[^96]: Instead of ext4 we will use xfs


# Test the configuration

`sudo mount -a`

# Verify that the whole set up is running

`df -h`![alt text](./Images/Screen%20Shot%202023-02-28%20at%2015.11.56.png)

# Install NFS server

`sudo yum -y update`

`sudo yum install nfs-utils -y`

`sudo systemctl start nfs-server.service`

`sudo systemctl enable nfs-server.service`

`sudo systemctl status nfs-server.service`

# Set up premission  that will permit the Web servers to read, write and execute files on NFS

`sudo chown -R nobody: /mnt/apps`

`sudo chown -R nobody: /mnt/logs`

`sudo chown -R nobody: /mnt/opt`

`sudo chmod -R 777 /mnt/apps`

`sudo chmod -R 777 /mnt/logs`

`sudo chmod -R 777 /mnt/opt`

# Restart the NFS server

`sudo systemctl restart nfs-server.service`

`sudo systemctl status nfs-server.service`

# Copy the Subnit CIDR of the nfs-server which can be found in the 'networking' tab of the NFS-server
 
 # Configure access to NFS for clients within the same subnet

 `sudo vi /etc/exports`

 `/mnt/apps 172.31.0.0/20(Subnet ID)(rw,sync,no_all_squash,no_root_squash)`

`/mnt/logs 172.31.0.0/20(rw,sync,no_all_squash,no_root_squash)`

`/mnt/opt172.31.0.0/20(rw,sync,no_all_squash,no_root_squash)`

`sudo exportfs -arv`

# Check which port is being used by NFS-Server and add it up to the security group

`rpcinfo -p | grep nfs`![alt text](./Images/Screen%20Shot%202023-02-28%20at%2015.33.02.png)

# Configure a DB server(using Ubuntu) and open port 3306 on the security group

`sudo apt update`

`sudo apt install mysql-server `

# Create a database 
on MYSQL-SERVER

Install MySQL server
Create a database and name it tooling
Create a database user and name it webaccess
Grant permission to webaccess user on tooling database to do anything only from the webservers subnet cidr

# Edit the bind address and change to 0.0.0.0
`sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf`

`sudo systemctl restart mysql`

# Prepare the webservers(2) using Redhat
1. Edit the inbound rule and add port 80

# Install NFS client to the webservers

`sudo yum install nfs-utils nfs4-acl-tools -y`

# Mount /var/www/ and target the NFS server’s export for apps

`sudo mkdir /var/www`

`sudo mount -t nfs -o rw,nosuid 172.31.59.39:/mnt/apps /var/www`

# Verify that the NFS was mounted successfully

`df -h`![alt text](./Images/Screen%20Shot%202023-03-04%20at%2010.38.53.png)

# Edit /etc/fstab

`sudo vi /etc/fstab`

Add `172.31.59.39(private address of the NFS server):/mnt/apps /var/www nfs defaults 0 0`


# Install Remi’s repository, Apache and PHP

`sudo yum install httpd -y`


`sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm`


`sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm`


`sudo dnf module reset php`


`sudo dnf module enable php:remi-7.4`


`sudo dnf install php php-opcache php-gd php-curl php-mysqlnd`


`sudo systemctl start php-fpm`


`sudo systemctl enable php-fpm`

Change to the root user

`sudo su`

`setsebool -P httpd_execmem 1`

# Verify that Apache files and directories are available on the Web Server in /var/www and also on the NFS server in /mnt/apps. If you see the same files – it means NFS is mounted correctly. You can try to create a new file touch test.txt from one server and check if the same file is accessible from other Web Servers.

`cd /var/www/html`

`touch proj`

`echo Hello >> proj`

`ls`

Check the other servers and see if you have the same file on them

`rm proj`

# Fork the tooling source code from Darey.io Github Account to your Github account. 

`sudo yum install git`

`git clone https://github.com/darey-io/tooling.git`

`ls` [^252]

[^252]: ls to view the fils and folders in the tooling folder

cd into tooling

`cd tooling`

`ls`

# Deploy the tooling website’s code to the Webserver. Ensure that the html folder from the repository is deployed to /var/www/html

cd back to html

`cd ..`

`mv tooling/* . [^268]`

[^268]: Move  everithing in tooling to html

`ls tooling`

`ls`

`sudo rm -r tooling` [^276]

[^276]: Remove tooling

`mv ./html/* .`

`ls html`

`ls`

`sudo rm -r html` [^286]

[^286]: Remove html

# Install MYSQL on the webservers

`sudo yum install mysql`

Update the website’s configuration to connect to the database (in /var/www/html/functions.php file)

`cd /var/www/html`

# Apply tooling-db.sql script to your database using this command mysql -h <database-private-ip> -u <db-username> -p< db-password> < tooling-db.sql

`vi functions.php`


`mysql -u webaccess -ppassword -h 172.31.59.22` on the webservers

cd back to /var/www/html

`mysql -u webaccess -ppassword -h 172.31.59.22 tooling < tooling-db.sql`

`mysql -u webaccess -ppassword -h 172.31.59.22 tooling`

`select * from users`

# Disable disable SELinux 

`sudo setenforce 0` in /var/www/html

`sudo vi /etc/sysconfig/selinux` and set SELINUX=disabled

`sudo systemctl restart httpd`

Open the website in your browser http://<Web-Server-Public-IP-Address-or-Public-DNS-Name>/index.php and make sure you can login into the website with myuser user.

![alt text](./Images/Screen%20Shot%202023-03-04%20at%2014.37.43.png)








































