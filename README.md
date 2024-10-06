# DEPLOYING A WORDPRESS WEB SOLUTION WITH A REMOTE MYSQL DATABASE ON AWS EC2 (RED HAT)

## INTRODUCTION

In this project, I deployed a **WordPress** website with a **MySQL** database, using **AWS EC2 instances** with Red Hat as the operating system. We utilized **three EBS volumes** for the **web server** (WordPress) and **one EBS volume** for the **database server** (MySQL). This documentation provides a step-by-step guide, including partitioning and configuring the EBS volumes for WordPress and MySQL, connecting the services, and addressing various errors encountered during the setup.

---

## PREREQUISITES

- AWS account
- Red Hat EC2 instances (one for the web server, one for the database server)
- EBS volumes attached to each instance
- Security group rules:
  - HTTP (port 80) for the web server
  - MySQL (port 3306) for the database server
  - SSH (port 22) for both servers
- SSH access configured with a key pair

---

## PROJECT ARCHITECTURE

- **Web Server**: EC2 instance with Red Hat, Apache, PHP, and WordPress installed.
- **Database Server**: EC2 instance with Red Hat and MySQL installed.
- **Storage**: Three EBS volumes attached to the web server for WordPress storage and one EBS volume attached to the database server for MySQL storage.

---

## STEP 1: LAUNCH AND CONFIGURE EC2 INSTANCES

### 1.1 Launch Web Server EC2 Instance

1. Launch a **Red Hat** EC2 instance.
2. Select the **t3.medium** instance type for better performance (upgraded from t3.micro).
3. Assign a security group with rules allowing inbound HTTP, SSH, and custom MySQL access.

![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/lauch%20a%20redhart%20linux%20server%20on%20aws.PNG)

### 1.2 Launch Database Server EC2 Instance

1. Launch another **Red Hat** EC2 instance.
2. Select the **t3.medium** instance type.
3. Assign a security group allowing MySQL access on port 3306 only from the web server’s private IP, and allow SSH access.

---
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/db%20create%20db%20server.PNG)

## : CONFIGURE EBS VOLUMES FOR EACH INSTANCE
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/created%20three%20volumes%20with%2010%20gib%20each.PNG)

### 2.1 Attach and Partition EBS Volumes for the Web Server
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/attach%20each%20volume.PNG)
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/attach%20each%20volume%20using%20the%20availabilty%20zone%20of%20the%20web%20server%20instance.PNG)

1. **Attach three EBS volumes** to the web server.
2. SSH into the web server instance:
   ```bash
   ssh -i "your-key.pem" ec2-user@<Web-Server-Public-IP>
   ```
   ![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/ssh%20into%20the%20redhart%20server%20using%20ec2-user.PNG)
3. **List the available disks** to check the attached volumes:
   ```bash
   lsblk
   ```

   ![sreenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/using%20lsblk%20to%20check%20the%20volumes.PNG)


### Create Partitions
 Use `gdisk` utility to create a single partition on each of the 3 disks

```bash
sudo gdisk /dev/nvme1n1
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/creating%20partitions.PNG)
Follow the prompts to create the partition, then use `w` to write changes. Exit out of the `gdisk` console and do the same for the remaining disks.

2. Verify partitions with:

```bash
lsblk
```   
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/partition%20created.PNG)

### Install LVM

1. Install `lvm2` package:

```bash
sudo yum install lvm2
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/installing%20lvm2.PNG)
2. Check for available partitions:

```bash
sudo lvmdiskscan
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/checking%20for%20available%20partition.PNG)

### Create Physical Volumes

Mark each of the 3 disks as physical volumes (PVs):

```bash
sudo pvcreate /dev/nvme1n1p1
sudo pvcreate /dev/nvme2n1p1
sudo pvcreate /dev/nvme3n1p1
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/Mark%20the%20partitions%20as%20physical%20volumes.PNG)
Verify creation:

```bash
sudo pvs
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/verify%20that%20the%20pysical%20partition%20has%20been%20created.PNG)

### Create or Extend a Volume Group

 To create a new volume group, use the following command:

```bash
sudo vgcreate webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/volume%20group%20webdata%20created%20for%20the%20three%20partition.PNG)

Verify VG creation:

```bash
sudo vgs
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/verify%20that%20VGhas%20been%20created.PNG)

### Create Logical Volumes

Create 2 logical volumes: `apps-lv` (use half of the PV size) and `logs-lv` (use the remaining space):

```bash
sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/create%20and%20verify%20that%20lvcreate%20forwebsite%20and%20logs.PNG)
Verify Logical Volume creation:

```bash
sudo lvs
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/create%20and%20verify%20that%20lvcreate%20forwebsite%20and%20logs.PNG)

### Verify Setup

```bash
sudo vgdisplay -v   # View complete setup - VG, PV, and LV.
sudo lsblk
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/verify%20and%20check%20the%20entire%20setup.PNG)

### Format Logical Volumes

Format the logical volumes with the ext4 filesystem:

```bash
sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/Format%20the%20logical%20volumes.PNG)


## Create Directories

Create directories to store website and log files:

```bash
sudo mkdir -p /var/www/html
sudo mkdir -p /home/recovery/logs
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/create%20htmland%20recovery%20logs.PNG)

### Mount Logical Volumes

1. Mount `/var/www/html` on `apps-lv` logical volume:

```bash
sudo mount /dev/webdata-vg/apps-lv /var/www/html/
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/Mount%20the%20logical%20volumes%20and%20rsync.PNG)
2. Backup log files before mounting:

```bash
sudo rsync -av /var/log/ /home/recovery/logs/
```

3. Mount `/var/log` on `logs-lv` logical volume:

```bash
sudo mount /dev/webdata-vg/logs-lv /var/log
```

4. Restore log files back:

```bash
sudo rsync -av /home/recovery/logs/ /var/log
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/Mount%20the%20logical%20volumes.PNG)


### Update /etc/fstab

Use the UUID of the device to update the `/etc/fstab` file:

```bash
sudo blkid
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/sudo%20blkid.PNG)

Edit `/etc/fstab`:

```bash
sudo vi /etc/fstab
```


2. Add the following lines (replace `<UUID>` with the actual UUID) for `/var/www/html` (for `apps-lv`) and `/var/log` (for `logs-lv`):

   ```bash
   UUID=608ae3b2-07d5-4e34-9cd3-669d8efda2eb /var/www/html ext4 defaults 0 0
   UUID=c34d3bfa-de12-4ca0-ae2c-4f12bc881925 /var/log ext4 defaults 0 0
   ```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/content%20of%20the%20sudo%20vi%20etcfstab.PNG)


### Test the Configuration

Reload the daemon and test the configuration:
. 
```bash
sudo mount -a
sudo systemctl daemon-reload
df -h   # Verify your setup.
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/Test%20the%20configuration%20and%20reload%20the%20daemon.PNG)

![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/verify%20set%20by%20running%20df%20-h.PNG)

This will show the available space and confirm the mounted locations.





####  We already Launched a RedHat EC2 Instance for the Database Server


     - Allow **MySQL (port 3306)** only for the **private IP** of the Web Server (to allow communication between them).

   - Attach a storage volume ( **10GiB or more**).
   - **Launch the instance** and connect to it via SSH:
     ```bash
     ssh -i /path/to/key.pem ec2-user@<Database-Server-Public-IP>
     ```
     ![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/db%20ssh%20into%20the%20db%20server.PNG)

You can Repeat the same steps as for the Web Server in configuring the one volume, partitioning and mounting, but instead of `apps-lv`, create `db-lv` and mount it to `/db` directory. You can as well just follow the steps all over again below: 

####  Configure the Database Volume
This step involves partitioning and configuring the attached storage volume.
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/db%20create%20and%20attach%20volume%20to%20the%20db%20server.PNG)

1. **List available storage volumes**:
   ```bash
   lsblk
   ```
   You should see the main volume (where the OS is installed) and the additional volume (the new one you'll use for the database).
   ![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/db%20listing%20availble%20volumes.PNG)

2. **Partition the volume** (using `gdisk`):
   Let's assume your new volume is `/dev/nvme1n1`.

   1. **Start the partitioning process**:
   ```bash
   sudo gdisk /dev/nvme1n1
   ```
   ![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/db%20create%20a%20partition%20for%20the%20volume%20on%20db.PNG)

   2. Inside the `gdisk` interface:
      - Press `n` to create a **new partition**.
      - Press Enter to accept the default partition number.
      - Press Enter to accept the default first and last sector (to use the entire disk).
      - Type `w` to write changes and exit.

3. **Create a physical volume**:
   ```bash
   sudo pvcreate /dev/nvme1n1p1
   ```

4. **Create a volume group**:
   ```bash
   sudo vgcreate  /dev/nvme1n1p1
   ```

6. **Format the logical volume** as ext4:
   ```bash
   sudo mkfs.ext4 /dev/nvme1n1p1
   ```
    ![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/db%20Format%20the%20New%20Partition.PNG)

7. **Mount the logical volume** to `/db`:
   1. Create a directory to mount the volume:
   ```bash
   sudo mkdir -p /db
   ```

   2. Mount the logical volume to the `/db` directory:
   ```bash
   sudo mount /dev/nvme1n1 /db
   ```
   ![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/db%20creating%20the%20db%20and%20mounting%20the%20partition%20to%20it.PNG)

8. **Ensure the volume mounts automatically on reboot**:
   - Get the UUID of the logical volume:
   ```bash
   sudo blkid
   ```
   ![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/db%20getting%20the%20UUID%20of%20the%20new%20partition.PNG)

   - Edit the `/etc/fstab` file:
   ```bash
   sudo vi /etc/fstab
   ```
   - Add the following line to mount the logical volume at boot:
   ```bash
   UUID=<UUID-of-db-lv> /db ext4 defaults 0 0
   ```
   Replace `<UUID-of-db-lv>` with the actual UUID obtained from the `blkid` command.
  ![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/db%20updating%20the%20content%20of%20sudo%20vi%20etcfstab.PNG)

   - Save and exit the file.

9. **Test the configuration**:
``
   ```bash
   sudo mount -a
   df -h
   ```
   ![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/db%20mounting%20and%20verify%20that%20mount%20works.PNG)


   This command will apply the changes and attempt to mount all filesystems listed in `/etc/fstab`. If there are any errors, the terminal will output the issue.

## Step 3: Install WordPress on your Web Server EC2

1. Update the repository:

```bash
sudo yum -y update
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/installing%20wordpress%20on%20web%20server.PNG)

2. Install `wget`, Apache, and its dependencies:

```bash
sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/installing%20wordpress%20on%20web%20server.PNG)

3. Start Apache:

```bash
sudo systemctl enable httpd
sudo systemctl start httpd
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/installing%20apache%20php%20and%20updating%20on%20webserver.PNG)

4. Install PHP and its dependencies:

```bash
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum module list php
sudo yum module reset php
sudo yum module enable php:remi-7.4
sudo yum install php php-opcache php-gd php-curl php-mysqlnd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/installing%20apache%20php%20and%20updating%20on%20webserver.PNG)

### Configure SELinux Policies

```bash
sudo chown -R apache:apache /var/www/html/wordpress
sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/setting%20permission%20for%20html%20wordpress%20and%20installing%20sql.PNG)

```bash
sudo setsebool -P httpd_can_network_connect=1
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/selium%20polciy.PNG)

6. Restart Apache:

```bash
sudo systemctl restart httpd
```

7. Download WordPress and copy it to `/var/www/html`:

```bash
mkdir wordpress
cd wordpress
sudo wget http://wordpress.org/latest.tar.gz
sudo tar -xzvf latest.tar.gz
sudo rm -rf latest.tar.gz
cp wordpress/wp-config-sample.php wordpress/wp-config.php
cp -R wordpress /var/www/html/
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/selium%20polciy.PNG)

## SET UP MYSQL ON THE DATABASE SERVER

###  Install MySQL

1. SSH into the **Database Server**:
   ```bash
   ssh -i "your-key.pem" ec2-user@<DB-Server-Public-IP>
   ```

2. **Install MySQL Server**:
   ```bash
   sudo yum install -y mysql-server
   ```
   ![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/db%20installing%20mysql.PNG)
   ![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/db%20mysql%20installed.PNG)


3. **Start MySQL** and enable it to start on boot:
   ```bash
   sudo systemctl start mysqld
   sudo systemctl enable mysqld
   ```
   ![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/db%20installing%20mysql.PNG)

### 4.2 Configure MySQL for Remote Access

1. Log in to MySQL as the root user:
   ```bash
   sudo mysql
   ```

2. Create a **WordPress database** and user:
   ```sql
   CREATE DATABASE wordpress;
   CREATE USER 'myuser'@'<Web-Server-Private-IP>' IDENTIFIED BY 'your password';
   GRANT ALL PRIVILEGES ON wordpress.* TO 'myuser'@'<Web-Server-Private-IP>';
   FLUSH PRIVILEGES;
   EXIT
   ```

![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/db%20creating%20database%20and%20user%20to%20web%20server%20private%20ip.PNG)

---

##  Configure WordPress to Connect to Remote Database

1. **Don't forget to Open MySQL port 3306** on the DB Server EC2.
   - For extra security, allow access to the DB server ONLY from your Web Server's IP address.

2. **Install MySQL client** on your Web Server:

```bash
sudo yum install mysql
```

3. **Test connection** from your Web Server to your DB server:

```bash
sudo mysql -u myuser -p -h <DB-Server-Private-IP-address>
```
![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/wb%20entering%20from%20webserver%20into%20db%20sql%20uing%20db%20ip.PNG)

Verify if you can execute the `SHOW DATABASES;` command and see a list of existing databases.

2. Navigate to WordPress setup via browser:
   ```bash
   http://<Web-Server-Public-IP>/wordpress
   ```
   ![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/access%20the%20wbe%20browser%20public%20ip%20adress%20wordpress.PNG)

3. Enter the MySQL database credentials:
   - Database Name: `wordpress`
   - Username: `myuser`
   - Password: `your password`
   - Database Host: `<DB-Server-Private-IP>`

   ![screenshot](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/setting%20up%20wordpress%20page.PNG)

### If you see this message - it means your WordPress has successfully connected to your remote MySQL database

![screenshot of WordPress connection](https://github.com/Prince-Tee/stegHub_wordpress/blob/main/screenshots%20from%20my%20local%20env/configuration%20worked.PNG)

---

## CONCLUSION

The deployment of a WordPress web solution with a remote MySQL database on AWS EC2 using Red Hat was successfully completed. This setup allows for scalable management of the web and database services, ensuring efficient performance and reliability. 
