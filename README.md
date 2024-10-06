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

### 1.2 Launch Database Server EC2 Instance

1. Launch another **Red Hat** EC2 instance.
2. Select the **t3.medium** instance type.
3. Assign a security group allowing MySQL access on port 3306 only from the web server’s private IP, and allow SSH access.

---

## STEP 2: CONFIGURE EBS VOLUMES FOR EACH INSTANCE

### 2.1 Attach and Partition EBS Volumes for the Web Server

1. **Attach three EBS volumes** to the web server.
2. SSH into the web server instance:
   ```bash
   ssh -i "your-key.pem" ec2-user@<Web-Server-Public-IP>
   ```

3. **List the available disks** to check the attached volumes:
   ```bash
   lsblk
   ```

4. Partition the disks `/dev/nvme1n1`, `/dev/nvme2n1`, and `/dev/nvme3n1`, and make made them part of an LVM (Logical Volume Manager) setup as indicated by the `TYPE="LVM2_member"`.

Next steps would be to create logical volumes and format them for use. Here's a summary of what you'll do next:

### Step 1: Verify the Physical Volumes

Check if the new partitions were added as physical volumes (PVs) in LVM:

```bash
sudo pvs
```

### Step 2: Create or Extend a Volume Group

 To create a new volume group, use the following command:

```bash
sudo vgcreate webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1
```

Or, if you're adding them to an existing volume group (`webdata-vg` in your case), you can extend the volume group:

```bash
sudo vgextend webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1
```

### Step 3: Create Logical Volumes

You can create logical volumes (e.g., for `apps`, `logs`, etc.). For example, to create a logical volume for storing application data:

```bash
sudo lvcreate -L 5G -n apps-lv webdata-vg
```

You can repeat the process for different logical volumes as needed.

### Step 4: Format the Logical Volumes

After creating the logical volumes, you need to format them. For example, to format `apps-lv` as ext4:

```bash
sudo mkfs.ext4 /dev/webdata-vg/apps-lv
```

You can replace `apps-lv` with other logical volumes you have created and format them as needed.

### Step 5: Mount the Logical Volumes

Create mount points and mount the logical volumes. For example:

```bash
sudo mkdir /mnt/apps
sudo mount /dev/webdata-vg/apps-lv /mnt/apps
```

To ensure the logical volumes are mounted at boot, you add them to the `/etc/fstab` file.

### Steps:
1. Open `/etc/fstab` for editing:

   ```bash
   sudo vi /etc/fstab
   ```

2. Add the following lines to mount the logical volumes for `/var/www/html` (for `apps-lv`) and `/var/log` (for `logs-lv`):

   ```bash
   UUID=608ae3b2-07d5-4e34-9cd3-669d8efda2eb /var/www/html ext4 defaults 0 0
   UUID=c34d3bfa-de12-4ca0-ae2c-4f12bc881925 /var/log ext4 defaults 0 0
   ```


manually mount the new partitions without rebooting:

   ```bash
   sudo mount -a
   ```
This will ensure that your logical volumes are mounted properly
### Step 6: Verify the Setup

Finally, verify that the logical volumes are correctly mounted:

```bash
df -h
```

This will show the available space and confirm the mounted locations.



### Step-by-Step Database Server Setup on RedHat EC2 Instance

#### Step 1: Launch a RedHat EC2 Instance for the Database Server
1. **Launch an EC2 instance**:
   - Go to the AWS Management Console > **EC2** > **Launch Instance**.
   - Select **Red Hat Enterprise Linux** as your Amazon Machine Image (AMI).
   - Choose **t2.micro** for the instance type (free-tier eligible).
   - Set the **security group**:
     - Allow **SSH (port 22)** for your IP.
     - Allow **MySQL (port 3306)** only for the **private IP** of the Web Server (to allow communication between them).
   - Attach a storage volume (let's assume **8GB or more**).
   - **Launch the instance** and connect to it via SSH:
     ```bash
     ssh -i /path/to/key.pem ec2-user@<Database-Server-Public-IP>
     ```

#### Step 2: Configure the Database Volume
This step involves partitioning and configuring the attached storage volume.

1. **List available storage volumes**:
   ```bash
   lsblk
   ```
   You should see the main volume (where the OS is installed) and the additional volume (the new one you'll use for the database).

2. **Partition the volume** (using `gdisk`):
   Let's assume your new volume is `/dev/nvme1n1`.

   1. **Start the partitioning process**:
   ```bash
   sudo gdisk /dev/nvme1n1
   ```

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
   sudo vgcreate dbdata-vg /dev/nvme1n1p1
   ```

5. **Create a logical volume** (we'll name it `db-lv`):
   ```bash
   sudo lvcreate -L 8G -n db-lv dbdata-vg
   ```
   Adjust the size (`-L 8G`) as needed.

6. **Format the logical volume** as ext4:
   ```bash
   sudo mkfs.ext4 /dev/dbdata-vg/db-lv
   ```

7. **Mount the logical volume** to `/db`:
   1. Create a directory to mount the volume:
   ```bash
   sudo mkdir -p /db
   ```

   2. Mount the logical volume to the `/db` directory:
   ```bash
   sudo mount /dev/dbdata-vg/db-lv /db
   ```

8. **Ensure the volume mounts automatically on reboot**:
   - Get the UUID of the logical volume:
   ```bash
   sudo blkid
   ```

   - Edit the `/etc/fstab` file:
   ```bash
   sudo vi /etc/fstab
   ```
   - Add the following line to mount the logical volume at boot:
   ```bash
   UUID=<UUID-of-db-lv> /db ext4 defaults 0 0
   ```
   Replace `<UUID-of-db-lv>` with the actual UUID obtained from the `blkid` command.

   - Save and exit the file.

9. **Test the configuration**:
   ```bash
   sudo mount -a
   ```

   This command will apply the changes and attempt to mount all filesystems listed in `/etc/fstab`. If there are any errors, the terminal will output the issue.

#### Step 3: Install and Configure MySQL
1. **Install MySQL Server**:
   ```bash
   sudo yum install -y mysql-server
   ```

2. **Start MySQL Service**:
   ```bash
   sudo systemctl start mysqld
   ```

3. **Enable MySQL Service on Boot**:
   ```bash
   sudo systemctl enable mysqld
   ```

4. **Secure MySQL Installation**:
   ```bash
   sudo mysql_secure_installation
   ```

5. **Access MySQL**:
   ```bash
   mysql -u root -p
   ```


## STEP 3: INSTALL AND CONFIGURE APACHE AND PHP ON THE WEB SERVER

### 3.1 Install Apache and PHP

1. SSH into the **Web Server**:
   ```bash
   ssh -i "your-key.pem" ec2-user@<Web-Server-Public-IP>
   ```

2. **Install Apache and PHP**:
   ```bash
   sudo yum -y update
   sudo yum -y install httpd php php-mysqlnd
   ```

3. **Start Apache** and enable it to start on boot:
   ```bash
   sudo systemctl start httpd
   sudo systemctl enable httpd
   ```

### 3.2 Download and Install WordPress

1. Navigate to the **Apache root directory**:
   ```bash
   cd /var/www/html
   ```

2. Download and extract WordPress:
   ```bash
   sudo wget http://wordpress.org/latest.tar.gz
   sudo tar -xzvf latest.tar.gz
   sudo rm -f latest.tar.gz
   ```

3. Move WordPress to its directory and set correct permissions:
   ```bash
   sudo mv wordpress /var/www/html/
   sudo chown -R apache:apache /var/www/html/wordpress
   sudo chmod -R 755 /var/www/html/wordpress
   ```

### Screenshot
> **[Add screenshot of Apache and WordPress installation]**

---

## STEP 4: SET UP MYSQL ON THE DATABASE SERVER

### 4.1 Install MySQL

1. SSH into the **Database Server**:
   ```bash
   ssh -i "your-key.pem" ec2-user@<DB-Server-Public-IP>
   ```

2. **Install MySQL**:
   ```bash
   sudo yum -y install mysql-server
   ```

3. **Start MySQL** and enable it to start on boot:
   ```bash
   sudo systemctl start mysqld
   sudo systemctl enable mysqld
   ```

### 4.2 Configure MySQL for Remote Access

1. Log in to MySQL as the root user:
   ```bash
   sudo mysql
   ```

2. Create a **WordPress database** and user:
   ```sql
   CREATE DATABASE wordpress;
   CREATE USER 'myuser'@'<Web-Server-Private-IP>' IDENTIFIED BY 'mypass';
   GRANT ALL PRIVILEGES ON wordpress.* TO 'myuser'@'<Web-Server-Private-IP>';
   FLUSH PRIVILEGES;
   ```

### Screenshot
> **[Add screenshot of MySQL setup]**

---

## STEP 5: CONNECT WORDPRESS TO THE REMOTE DATABASE

1. Test the MySQL connection from the web server:
   ```bash
   mysql -u myuser -p -h <DB-Server-Private-IP>
   ```

2. Navigate to WordPress setup via browser:
   ```bash
   http://<Web-Server-Public-IP>/wordpress
   ```

3. Enter the MySQL database credentials:
   - Database Name: `wordpress`
   - Username: `myuser`
   - Password: `mypass`
   - Database Host: `<DB-Server-Private-IP>`

### Screenshot
> **[Add screenshot of WordPress connection setup]**

---

## STEP 6: COMPLETE WORDPRESS INSTALLATION

Follow the instructions to complete the installation, set the site name, admin credentials, and log in to the Word

Press dashboard.

### Screenshot
> **[Add screenshot of completed WordPress setup]**

---


## CONCLUSION

The deployment of a WordPress web solution with a remote MySQL database on AWS EC2 using Red Hat was successfully completed. This setup allows for scalable management of the web and database services, ensuring efficient performance and reliability. 
