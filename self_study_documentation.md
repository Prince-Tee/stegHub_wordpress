### Self-Study Documentation: Issues and Resolutions for EC2 WordPress Deployment with MySQL

This document highlights the challenges encountered while deploying a WordPress website on an AWS EC2 instance using a MySQL database server. It also outlines the step-by-step resolutions applied to resolve these issues.

---

### **Problems Faced and Solutions**

#### 1. **Setting SELinux Boolean Value**
**Issue:**
When attempting to set the SELinux Boolean value with the command:

```bash
setsebool -P httpd_execmem 1
```

I encountered the following error:
```
Cannot set persistent booleans, please try as root.
```

**Solution:**
To solve this, I used the `sudo` command to run the command with elevated privileges:
```bash
sudo setsebool -P httpd_execmem 1
```

This allowed me to modify the SELinux settings without any issues.

---

#### 2. **Performance Issue with Dependencies Stuck**
**Issue:**
After deploying the WordPress and MySQL setup, the server was lagging, and dependencies seemed to be stuck.

**Analysis:**
Upon checking, I realized that the instance type was too small (t3.micro) to handle the workload effectively.

**Solution:**
I modified the EC2 instance type from `t3.micro` to `t3.medium` to increase CPU and memory resources. After this change, the performance issues were resolved, and the dependencies stopped lagging and hanging.

**Steps to Modify EC2 Instance Type:**
1. Stop the EC2 instance:
   ```bash
   sudo systemctl stop httpd
   ```
2. Navigate to the **EC2 Management Console** on AWS.
3. Right-click the instance, and choose **Instance Settings > Change Instance Type**.
4. Select **t3.medium** from the available instance types.
5. Start the instance again.

This change provided more resources to the server and resolved the performance issues.

---

#### 3. **Accidentally Downloaded WordPress on Database Server**
**Issue:**
I mistakenly downloaded WordPress on the MySQL database server instead of the web server. When trying to access the MySQL database, I encountered the following error:

```bash
ERROR 1130 (HY000): Host 'ip-172-31-32-231.eu-north-1.compute.internal' is not allowed to connect to this MySQL server.
```

**Solution:**
To resolve the MySQL connection issue, I had to configure MySQL to allow connections from external IP addresses.

**Steps Taken:**
1. I connected to the MySQL database server and updated the MySQL user privileges using the following command:
   ```sql
   GRANT ALL PRIVILEGES ON *.* TO 'myuser'@'%' IDENTIFIED BY 'password';
   FLUSH PRIVILEGES;
   ```

2. Restarted the MySQL server to apply changes:
   ```bash
   sudo systemctl restart mysql
   ```

3. After resolving the MySQL connection issue, I removed WordPress from the database server and downloaded it on the correct web server.

---

#### 5. **Accessing the Web Server without Editing wp-config.php**
**Observation:**
Normally, after setting up WordPress, one must update the `wp-config.php` file with the database credentials (e.g., database name, username, password, and host). However, in this case, I was able to access the WordPress setup page via the web browser without manually editing the `wp-config.php` file.

**Reason:**
WordPress has an automated setup that tries to detect the database and prompts you to enter the credentials through the web interface. I entered the necessary details directly through the WordPress setup wizard in the browser, and the configuration was automatically updated.

**Steps:**
1. Accessed the web server through the public IP:
   ```bash
   http://<Web-Server-Public-IP>/wordpress/
   ```
2. WordPress prompted me to enter the database credentials directly in the browser setup interface.
3. After providing the correct details, the configuration was saved without the need to manually edit the `wp-config.php` file.

---

### **Summary of Key Lessons Learned**

1. **Importance of Instance Size:**
   For performance-heavy workloads like running a WordPress and MySQL database server, using a small instance type such as `t3.micro` can cause lagging and dependency issues. Upgrading the instance type to `t3.medium` provided the necessary resources to run the server smoothly.

2. **Sudo Privileges for System Commands:**
   Certain system commands, such as setting SELinux policies and restarting services, require root privileges. Using `sudo` ensures that commands are executed with the necessary permissions.

3. **WordPress Auto-Configuration:**
   The WordPress setup wizard in the browser can automatically handle `wp-config.php` configurations during the installation process. This simplifies the deployment process by removing the need for manual file editing.

---