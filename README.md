# Web-Solution-With-WordPress
Step 1 — Prepare a Web Server
Launch an EC2 instance that will serve as "Web Server". Create 3 volumes in the same AZ as your Web Server EC2.
Attach all three volumes one by one to your Web Server EC2 instance.
Open up the Linux terminal to begin configuration
Use lsblk command to inspect what block devices are attached to the server. Notice names of your newly created devices. All devices in Linux reside in /dev/ directory. Inspect it with ls /dev/ and make sure you see all 3 newly created block devices there - their names will likely be xvdb, xvdc, xvdd.

![image](https://github.com/user-attachments/assets/96fc98dd-d2eb-4fe4-878f-97966f803dd9)

Use gdisk utility to create a single partition on each of the 3 disks
sudo gdisk /dev/xvdf
GPT fdisk (gdisk) version 1.0.3

Partition table scan:
  MBR: not present
  BSD: not present
  APM: not present
  GPT: not present

Creating new GPT entries.

Command (? for help branch segun-edits: p
Disk /dev/xvdf: 20971520 sectors, 10.0 GiB
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): D936A35E-CE80-41A1-B87E-54D2044D160B
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 20971486
Partitions will be aligned on 2048-sector boundaries
Total free space is 2014 sectors (1007.0 KiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048        20971486   10.0 GiB    8E00  Linux LVM

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): yes
OK; writing new GUID partition table (GPT) to /dev/xvdf.
The operation has completed successfully.
Now,  your changes has been configured succesfuly, exit out of the gdisk console and do the same for the remaining disks.

Use lsblk utility to view the newly configured partition on each of the 3 disks
## lsblk

![image](https://github.com/user-attachments/assets/0e60327a-f74d-40c7-acb6-d45e56b00720)

.Install lvm2 package using sudo yum install lvm2. 
Run sudo lvmdiskscan command to check for available partitions.
![image](https://github.com/user-attachments/assets/18ff49d3-e919-4be0-8628-1a21ecc5f980)

Use pvcreate utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM
sudo pvcreate /dev/xvdb1
sudo pvcreate /dev/xvdc1
sudo pvcreate /dev/xvdd1
Verify that your Physical volume has been created successfully by running sudo pvs.

![image](https://github.com/user-attachments/assets/732483fb-7ba2-42f3-9dd4-eae069c58e00)

Use vgcreate utility to add all 3 PVs to a volume group (VG). Name the VG webdata-vg
sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1
Verify that your VG has been created successfully by running sudo vgs

![image](https://github.com/user-attachments/assets/af8d4ea5-39ba-4043-b73b-87eba85eefe5)

store data for logs.
sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg

Verify that your Logical Volume has been created successfully by running sudo lvs
Use mkfs.ext4 to format the logical volumes with ext4 filesystem
sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv

Create /var/www/html directory to store website files sudo mkdir -p /var/www/html
Create /home/recovery/logs to store backup of log data sudo mkdir -p /home/recovery/logs
Mount /var/www/html on apps-lv logical volume
sudo mount /dev/webdata-vg/apps-lv /var/www/html/
Use rsync utility to backup all the files in the log directory /var/log into /home/recovery/logs (This is required before mounting the file system)
sudo rsync -av /var/log/ /home/recovery/logs/
Mount /var/log on logs-lv logical volume. (Note that all the existing data on /var/log will be deleted. That is why step 15 above is very important)
sudo mount /dev/webdata-vg/logs-lv /var/log
Restore log files back into /var/log directory
sudo rsync -av /home/recovery/logs/ /var/log
Verify the entire setup
sudo vgdisplay -v #view complete setup - VG, PV, and LV
sudo lsblk

![image](https://github.com/user-attachments/assets/25d141b6-3149-4bc4-86dc-7f3a8afa4cc7)


Update /etc/fstab file so that the mount configuration will persist after restart of the server.
The UUID of the device will be used to update the /etc/fstab file;
sudo blkid

![image](https://github.com/user-attachments/assets/324741be-8b4a-448b-bb55-6fb770d08698)

sudo vi /etc/fstab
Update /etc/fstab in this format using your own UUID and rememeber to remove the leading and ending quotes.


![image](https://github.com/user-attachments/assets/d7a4a085-6281-4c0b-a3f1-8439e12383f4)


Test the configuration and reload the daemon
sudo mount -a
sudo systemctl daemon-reload
Verify your setup by running df -h, 
Step 2 — Prepare the Database Server
Launch a second RedHat EC2 instance that will have a role - 'DB Server' Repeat the same steps as for the Web Server, but instead of apps-lv create db-lv and mount it to /db directory instead of /var/www/html/.
After repeating the process, your output will be like this.

![image](https://github.com/user-attachments/assets/d24e40c0-ac49-40f8-a949-dd0e90ee1b9d)

Step 3 — Install WordPress on your Web Server EC2
Update the repository
sudo yum -y update
Install wget, Apache and it's dependencies
sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
Start Apache
sudo systemctl enable httpd sudo systemctl start httpd
To install PHP and it's dependencies
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum module list php
sudo yum module reset php
sudo yum module enable php:remi-7.4
sudo yum install php php-opcache php-gd php-curl php-mysqlnd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
setsebool -P httpd_execmem 1
Restart Apache
sudo systemctl restart httpd

![image](https://github.com/user-attachments/assets/74584677-7256-4dde-8fb6-dab0e3a4d867)

Download wordpress and copy wordpress to /var/www/html
mkdir wordpress
cd wordpress
sudo wget http://wordpress.org/latest.tar.gz
sudo tar -xzvf latest.tar.gz
sudo rm -rf latest.tar.gz
cp wordpress/wp-config-sample.php wordpress/wp-config.php
cp -R wordpress /var/www/html/
Configure SELinux Policies
sudo chown -R apache:apache /var/www/html/wordpress
sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
sudo setsebool -P httpd_can_network_connect=1
Use your browser to access the public IP of your webserver,if successful you will the red hat enterprise linux page.

![image](https://github.com/user-attachments/assets/5127275c-c781-4cfd-902e-81082939201d)

Step 4 — Install MySQL on your DB Server EC2
sudo yum update
sudo yum install mysql-server
Verify that the service is up and running by using sudo systemctl status mysqld, if it is not running, restart the service and enable it so it will be running even after reboot:
sudo systemctl restart mysqld
sudo systemctl enable mysqld
Step 5 — Configure DB to work with WordPress
sudo mysql
CREATE DATABASE wordpress;
CREATE USER 'myuser'@'172.31.42.113' IDENTIFIED BY 'mypass';


GRANT ALL ON wordpress.* TO 'myuser'@'172.31.42.113';
FLUSH PRIVILEGES;
SHOW DATABASES;
exit

![image](https://github.com/user-attachments/assets/2fa0f05b-ad8e-43d0-9159-f1727faad28e)

Step 6 — Configure WordPress to connect to remote database.
Hint: Do not forget to open MySQL port 3306 on DB Server EC2. For extra security, you shall allow access to the DB server ONLY from your Web Server's IP address, so in the Inbound Rule configuration specify source as /32

![image](https://github.com/user-attachments/assets/ca79cb4c-060f-479d-a608-996a37dd8cf9)

Install MySQL client and test that you can connect from your Web Server to your DB server by using mysql-client
sudo yum install mysql
sudo mysql -u admin -p -h <DB-Server-Private-IP-address>

![image](https://github.com/user-attachments/assets/5e7380a4-9cb4-4cab-8886-80fe563224ad)

Verify if you can successfully execute SHOW DATABASES; command and see a list of existing databases

![image](https://github.com/user-attachments/assets/cf77e84a-9898-4d05-bc99-56c564ddb8bc)

Change permissions and configuration so Apache could use WordPress:
sudo find //var/www/htm/ -type f -exec chmod 644 {} \;
sudo find //var/www/htm/ -type d -exec chmod 755 {} \;
sudo chmod 440 /var/www/html/wp-config.php
Verify wp-config.php Settings
Open your wp-config.php file located in the root of your WordPress installation.
Ensure the database credentials (DB_NAME, DB_USER, DB_PASSWORD, DB_HOST) are correct.
Sudo vi /var/www/html/wp-config.php
define('DB_NAME', 'your_database_name');
define('DB_USER', 'your_username');
define('DB_PASSWORD', 'your_password');
define('DB_HOST', 'localhost'); // or the correct host

![image](https://github.com/user-attachments/assets/bf084953-099b-4e13-af1a-64546eb6c3c4)

Enable TCP port 80 in Inbound Rules configuration for your Web Server EC2 (enable from everywhere 0.0.0.0/0 or from your workstation's IP)
Disable the Apache default page:the default page can be renamed.
sudo mv /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/welcome.conf_backup

![image](https://github.com/user-attachments/assets/bb3db571-dc02-42ec-9af2-4f25f0c012fd)

Try to access from your browser the link to your WordPress http://<Web-Server-Public-IP-Address>/wordpress/
![image](https://github.com/user-attachments/assets/58386e6d-0525-4b65-8e64-ba35575e99dc)

If you see this message - it means your WordPress has successfully connected to your remote MySQL database

![image](https://github.com/user-attachments/assets/5a90724a-d5d3-4901-9adf-d2d9a0061881)
