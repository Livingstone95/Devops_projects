#Devops Tooling Website Solution, A PBL Progressive Project from [Darey.io](https://darey.io/)

###The setups is based on;
1. Infrastructure: VmWare Player
2. Webserver Linux Distro: Centos
3. Database Server Linux Distro: Ubuntu
4. Storage: NFS
5. 5Programming Language: PHP
6. Code Repository: GitHub





##Step 1; Preparing the NFS Server:

change to root user
`sudo su -`
    
1. **check all disk availabe on the machine**
`lsblk`
![listing all the blocks available]("C:\Users\LIVINGSTONE\devops_git\Devops_project\images\lsblk.png")

2. ***To create physical volumes with the the disk***
`pvcreate /dev/sdb /dev/sdc /dev/sdd`

*To create Volume groups for apps, logs and opt*
`vgcreate vg-apps /dev/sdb`
`vgcreate vg-logs /dev/sdc`
`vgcreate vg-opt /dev/sdd`

*To create logical volumes for apps, logs and opt*.
`lvcreate -l 100%FREE -n lv-apps vg-apps`
`lvcreate -l 100%FREE -n lv-logs vg-logs`
`lvcreate -l 100%FREE -n lv-opt vg-opt` 

3.***To mount the logical volumes***
> To mount the logical volumnes, a mount point will be created first, then it will be formated with a filesystem. For this project it was dformatted in `xfs` filesystem.

make directory for html
`mkdir /mnt/html`  

make directory for logs
`mkdir /mnt/logs`  

*Formatiing all Logical volumes to an `xfs` filesystem*
`mkfs.xfs /dev/vg-apps/lv-apps`
`mkfs.xfs /dev/vg-logs/lv-logs` 
`mount /dev/vg-apps/lv-apps` 
`mount /dev/vg-apps/lv-apps`

*To confirm the mounts*
`df -h`

4. **To install and configure NFS server**
install nfs server
`dnf install nfs-utils -y`
start nfs server
`systemctl start nfs-server.service`
enable the nfs server to boot on startup
`systemctl enable nfs-server.service` 
confirm the status of nfs server
`systemctl status nfs-server.service`
 
######To exports the mount for the webservers
>To avoid file restrictions on the NFS share directory, it’s advisable to configure directory ownership as shown. This allows creation of files from the client systems without encountering any permission issues.

`chown -R nobody: /mnt/html`
`chmod -R 777 /mnt/html`

`chown -R nobody: /mnt/logs`
`chmod -R 777 /mnt/logs`

*To edit the `/etc/exports` and paste the following*
`nano /etc/exports`

>                             /etc/fstab
    /mnt/html 192.168.233.0/24(rw,sync,no_all_squash,root_squash)
    /mnt/logs 192.168.233.0/24(rw,sync,no_all_squash,root_squash)

*for the cahnges to be effected, the nfs servcer must be restarted*
`systemctl restart nfs-utils.service`

*to export the files that has been created*
`exportfs -arv`
*to display exported files*
`exportfs -s`

*configure the firewall to allow clients*
>`firewall-cmd --permanent --add-service=nfs`
`firewall-cmd --permanent --add-service=rpc-bind`
`firewall-cmd --permanent --add-service=mountd`
`firewall-cmd --reload`

*To ensure that all the configuration is persistent after reboot this was pasted in the `/etc/fstab` folder*

`sudo nano /etc/fstab`
>                            /etc/fstab
    UUID=bcf9c9e6-191c-4e23-aad8-72d8f9415c2e /mnt/lv-apps xfs defaults,nofail 0 0
    UUID=94c434a5-3fbf-4f22-8ad9-75072ceb9c3b /mnt/lv-logs xfs defaults,nofail 0 0
    UUID=24ae87df-68b3-43e5-9f3e-2febb6ba7a0e /mnt/lv-opt xfs defaults,nofail 0 0

##Step 2; Configuring The Database

*installing mysql-server*
`dnf install mysql-server`

*start mysql service*
`systemctl start mysqld.service`
*confirm the status of mysql-server*
`systemctl status mysqld`
*to enablke mysql-server on startup*
`systemctl enable mysqld`

*To secure mkysql installation*
`mysql_secure_installation`
> answer the questions prompted by the server

***to create a database and a user***
*to login to mysql server as a root user*
`mysql -u root -p`

create database tooling and assign privinleges
>`CREATE USER 'webaccess'@'192.168.233.%' IDENTIFIED BY 'david';`
`GRANT ALL PRIVILEGES ON * . * TO 'webaccess'@'192.168.233.%';`

##Step 3: Preparing The webservers

1. *install nfs-client*
`dnf install nfs-utils nfs4-acl-tools -y`

to show the availabale exports folder in the NFS storage
`showmount -e 192.168.233.136`

2. *to mount `/var/wwww/html`  targeting the export for apps*

`mount -t nfs 192.168.233.136:/mnt/html /var/www`

3. *install apache*
`dnf install httpd`

*start httpd server*
   `systemctl start httpd`

*configure the firewall*
>`firewall-cmd --permanent --add-service=http`
`firewall-cmd --permanent --list-all`
`firewall-cmd –reload`

4.*The log folder for apache is `/var/log/httpd`*

5. >I followed the steps in the this [link](https://youtu.be/f5grYMXbAV0) to fork a repository.

6. >To deploy the tooling website to the webserver
after forking the tooling repository to my github account.

To clone the repository to my local machine

`git clone https://github.com` 
`mkdir /var/www/html/tooling`

copy the html folder of from the clone repository to the `/vwr/www/html/tooling` folder on my webserver which the is root folder for apache
`rsync -avz --no-o --no-g --no-perms /tooling/tooling/html/* /var/www/html/tooling`

reload the httpd service
`systemctl resart httpd`
