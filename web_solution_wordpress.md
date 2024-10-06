Sure! Here’s a step-by-step Markdown documentation for setting up a WordPress web solution on AWS using a 3-tier architecture with EC2 instances running RedHat Linux.

---

# Web Solution With WordPress

## Your 3-Tier Setup

1. A Laptop or PC to serve as a client.
2. An EC2 Linux Server as a web server (This is where you will install WordPress).
3. An EC2 Linux server as a database (DB) server.
   
**Use RedHat OS for this project.**

> **Note:** For this project, you will use the `ec2-user` for SSH connections. The connection string will look like:  
> `ssh ec2-user@<Public-IP>`

## Step 1: Prepare the Web Server

### Launch the EC2 Instance

1. Launch an EC2 instance that will serve as the "Web Server".
2. Create 3 volumes in the same Availability Zone (AZ) as your Web Server EC2, each of 10 GiB.
   - [Learn How to Add EBS Volume to an EC2 instance here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-attaching-volume.html).

### Attach Volumes

1. Attach all three volumes one by one to your Web Server EC2 instance.
2. Open the Linux terminal to begin configuration.

### Inspect Block Devices

Run the following commands:

```bash
lsblk   # To inspect block devices attached to the server.
ls /dev/  # To view all devices; your new devices should be xvdf, xvdh, xvdg.
df -h   # To see all mounts and free space on your server.
```

### Create Partitions

1. Use `gdisk` utility to create a single partition on each of the 3 disks:

```bash
sudo gdisk /dev/xvdf
```

> Follow the prompts to create the partition, then use `w` to write changes. Exit out of the `gdisk` console and do the same for the remaining disks.

2. Verify partitions with:

```bash
lsblk
```

### Install LVM

1. Install `lvm2` package:

```bash
sudo yum install lvm2
```

2. Check for available partitions:

```bash
sudo lvmdiskscan
```

### Create Physical Volumes

Mark each of the 3 disks as physical volumes (PVs):

```bash
sudo pvcreate /dev/xvdf1
sudo pvcreate /dev/xvdg1
sudo pvcreate /dev/xvdh1
```

Verify creation:

```bash
sudo pvs
```

### Create Volume Group

Add all 3 PVs to a volume group (VG). Name the VG `webdata-vg`:

```bash
sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1
```

Verify VG creation:

```bash
sudo vgs
```

### Create Logical Volumes

Create 2 logical volumes: `apps-lv` (use half of the PV size) and `logs-lv` (use the remaining space):

```bash
sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg
```

Verify Logical Volume creation:

```bash
sudo lvs
```

### Verify Setup

```bash
sudo vgdisplay -v   # View complete setup - VG, PV, and LV.
sudo lsblk
```

### Format Logical Volumes

Format the logical volumes with the ext4 filesystem:

```bash
sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```

### Create Directories

Create directories to store website and log files:

```bash
sudo mkdir -p /var/www/html
sudo mkdir -p /home/recovery/logs
```

### Mount Logical Volumes

1. Mount `/var/www/html` on `apps-lv` logical volume:

```bash
sudo mount /dev/webdata-vg/apps-lv /var/www/html/
```

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

### Update /etc/fstab

Use the UUID of the device to update the `/etc/fstab` file:

```bash
sudo blkid
```

Edit `/etc/fstab`:

```bash
sudo vi /etc/fstab
```

Add the following lines (replace `<UUID>` with the actual UUID):

```
<UUID>  /var/www/html  ext4  defaults  0  0
<UUID>  /var/log       ext4  defaults  0  0
```

### Test the Configuration

Reload the daemon and test the configuration:
. 
```bash
sudo mount -a
sudo systemctl daemon-reload
df -h   # Verify your setup.
```

## Step 2: Prepare the Database Server

1. Launch a second RedHat EC2 instance as the 'DB Server'.
2. Repeat the same steps as for the Web Server, but instead of `apps-lv`, create `db-lv` and mount it to `/db` directory.

## Step 3: Install WordPress on your Web Server EC2

1. Update the repository:

```bash
sudo yum -y update
```

2. Install `wget`, Apache, and its dependencies:

```bash
sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
```

3. Start Apache:

```bash
sudo systemctl enable httpd
sudo systemctl start httpd
```

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

5. Set SELinux policies:

```bash
setsebool -P httpd_execmem 1
```

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

### Configure SELinux Policies

```bash
sudo chown -R apache:apache /var/www/html/wordpress
sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
sudo setsebool -P httpd_can_network_connect=1
```

## Step 4: Install MySQL on your DB Server EC2

1. Update the repository:

```bash
sudo yum update
```

2. Install MySQL server:

```bash
sudo yum install mysql-server
```

3. Verify that the service is running:

```bash
sudo systemctl status mysqld
```

4. If it is not running, restart the service:

```bash
sudo systemctl restart mysqld
sudo systemctl enable mysqld
```

## Step 5: Configure DB to work with WordPress

1. Access MySQL:

```bash
sudo mysql
```

2. Create a database and user:

```sql
CREATE DATABASE wordpress;
CREATE USER 'myuser'@'<Web-Server-Private-IP-Address>' IDENTIFIED BY 'mypass';
GRANT ALL ON wordpress.* TO 'myuser'@'<Web-Server-Private-IP-Address>';
FLUSH PRIVILEGES;
SHOW DATABASES;
EXIT;
```

## Step 6: Configure WordPress to Connect to Remote Database

1. **Open MySQL port 3306** on the DB Server EC2.
   - For extra security, allow access to the DB server ONLY from your Web Server's IP address.

2. **Install MySQL client** on your Web Server:

```bash
sudo yum install mysql
```

3. **Test connection** from your Web Server to your DB server:

```bash
sudo mysql -u myuser -p -h <DB-Server-Private-IP-address>
```

4. Verify if you can execute the `SHOW DATABASES;` command and see a list of existing databases.

### Configure Apache Permissions

1. Enable TCP port 80 in Inbound Rules for your Web Server EC2.
2. Try to access WordPress in your browser:

```plaintext
http://<Web-Server-Public-IP-Address>/wordpress/
```

3. Fill out your DB credentials in the WordPress installation form.

> If you see a message indicating successful connection to your MySQL database, the setup is complete!

---

