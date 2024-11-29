# E-Commerce Application Deployment on CentOS

This is a guide to deploying a simple e-commerce application on CentOS systems, built for learning purposes. The deployment involves setting up a multi-node environment where one server hosts the database and the other serves the web application. The Repo is based on this tutorial [KodeKloud DevOps Prerequisites Course](https://github.com/kodekloudhub/learning-app-ecommerce)

![deployment](https://github.com/user-attachments/assets/d9575cd7-54ae-4e76-8841-6a9653a12889)


## Table of Contents

1. [Deploy Pre-Requisites](#1-deploy-pre-requisites)
2. [Deploy and Configure Database](#2-deploy-and-configure-database)
3. [Load Product Inventory Information to Database](#3-load-product-inventory-information-to-database)
4. [Deploy and Configure Web Server](#4-deploy-and-configure-web-server)
5. [Download the Code](#5-download-the-code)
6. [Create and Configure the .env File](#6-create-and-configure-the-env-file)
7. [Update index.php to Use Environment Variables](#7-update-indexphp-to-use-environment-variables)
8. [Test the Connection](#8-test-the-connection)
9. [Common Issues Faced](#9-common-issues-faced)
10. [Commands to View Database from Command Line](#10-commands-to-view-database-from-command-line)

---

## 1. **Deploy Pre-Requisites**

### Install FirewallD

FirewallD is required to control network traffic and secure the system. Install and enable it using the following commands:

```bash
sudo yum install -y firewalld
sudo systemctl start firewalld
sudo systemctl enable firewalld
sudo systemctl status firewalld
```
## 2. **Deploy and Configure Database**
The database for the e-commerce application is MariaDB. Follow these steps to install and configure it.

### Install MariaDB Server
```
sudo yum install -y mariadb-server
sudo systemctl start mariadb
sudo systemctl enable mariadb
```
### Configure MariaDB
Start the MariaDB service and configure it:
```
sudo vi /etc/my.cnf
sudo systemctl restart mariadb
```
### Configure Firewall for Database
Open port 3306 (MySQL default port) in the firewall to allow connections to the database server:
```
sudo firewall-cmd --permanent --zone=public --add-port=3306/tcp
sudo firewall-cmd --reload
```
### Configure Database and User
Login to MariaDB and create the necessary database and user for the application:
```
$ mysql
MariaDB > CREATE DATABASE ecomdb;
MariaDB > CREATE USER 'ecomuser'@'web-server-ip' IDENTIFIED BY 'ecompassword';
MariaDB > GRANT ALL PRIVILEGES ON *.* TO 'ecomuser'@'web-server-ip';
MariaDB > FLUSH PRIVILEGES;
```
For a multi-node setup, replace localhost with the IP address of the web server where PHP is running(e.g., 'ecomuser'@'192.168.55.x').

## 3. **Load Product Inventory Information to Database**
Create a SQL script to populate the database with some initial product information.

### Create the db-load-script.sql
```
cat > db-load-script.sql <<-EOF
USE ecomdb;
CREATE TABLE products (id mediumint(8) unsigned NOT NULL auto_increment, Name varchar(255) default NULL, Price varchar(255) default NULL, ImageUrl varchar(255) default NULL, PRIMARY KEY (id)) AUTO_INCREMENT=1;

INSERT INTO products (Name, Price, ImageUrl) VALUES ("Laptop", "100", "c-1.png"), ("Drone", "200", "c-2.png"), ("VR", "300", "c-3.png"), ("Tablet", "50", "c-5.png"), ("Watch", "90", "c-6.png"), ("Phone Covers", "20", "c-7.png"), ("Phone", "80", "c-8.png"), ("Laptop", "150", "c-4.png");

EOF
```
### Run the SQL Script
```
sudo mysql < db-load-script.sql
```
## 4. **Deploy and Configure Web Server**
The web server will host the PHP application. We will use Apache HTTP Server (httpd) along with PHP and the MySQLnd extension.

### Install Required Packages
Install the necessary packages for Apache, PHP, and MySQL support:

```
sudo yum install -y httpd php php-mysqlnd
```
### Install FirewallD

FirewallD is required to control network traffic and secure the system. Install and enable it using the following commands:

```bash
sudo yum install -y firewalld
sudo systemctl start firewalld
sudo systemctl enable firewalld
sudo systemctl status firewalld
```

### Configure Firewall for HTTP

Allow HTTP traffic by opening port 80 on the firewall:

```
sudo firewall-cmd --permanent --zone=public --add-port=80/tcp
sudo firewall-cmd --reload
```
### Configure Apache
Change the default DirectoryIndex to make index.php the default page:

```
sudo sed -i 's/index.html/index.php/g' /etc/httpd/conf/httpd.conf
```
### Start Apache HTTP Server
Start and enable the Apache service to start on boot:

```
sudo systemctl start httpd
sudo systemctl enable httpd
```
## 5. **Download the Code**
Download the e-commerce application code on the web server node using Git:

```
sudo yum install -y git
sudo git clone https://github.com/kodekloudhub/learning-app-ecommerce.git /var/www/html/
```
6. Create and Configure the .env File
In the root directory of the project, create a .env file to store the database connection information.

```
cat > /var/www/html/.env <<-EOF
DB_HOST=web-server-ip
DB_USER=ecomuser
DB_PASSWORD=ecompassword
DB_NAME=ecomdb
EOF
```
For multi-node setups, modify the DB_HOST value in the .env file to the IP address of the database server:

```
DB_HOST=192.168.x.x  # Database server IP address
```
## 7. **Update index.php to Use Environment Variables**
Modify index.php to load the environment variables from the .env file and use them for the database connection:

```
<?php
// Function to load environment variables from a .env file
function loadEnv($path)
{
    if (!file_exists($path)) {
        return false;
    }

    $lines = file($path, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);
    foreach ($lines as $line) {
        if (strpos(trim($line), '#') === 0) {
            continue;
        }

        list($name, $value) = explode('=', $line, 2);
        $name = trim($name);
        $value = trim($value);
        putenv(sprintf('%s=%s', $name, $value));
    }
    return true;
}

// Load environment variables from .env file
loadEnv(__DIR__ . '/.env');

// Retrieve the database connection details from environment variables
$dbHost = getenv('DB_HOST');
$dbUser = getenv('DB_USER');
$dbPassword = getenv('DB_PASSWORD');
$dbName = getenv('DB_NAME');
?>
```
## 8. **Test the Connection**
Test the connection to the web application by running the following curl command:

```
curl http://localhost
```
## 9. **Common Issues Faced**
### Issue: Permission Denied
While trying to connect the PHP web server to the MySQL database server, we encountered the error:
```
Connection failed: Permission denied
```
This occurred because the SELinux security policies were preventing the web server from connecting to the database.

### Solution
The solution was to enable the `httpd_can_network_connect_db` SELinux boolean, which allows the web server to connect to a remote database.

To fix this:
1. Check the current status of the SELinux boolean with:
    ```bash
    getsebool -a | grep httpd
    ```
2. If `httpd_can_network_connect_db` is off, enable it with:
    ```bash
    setsebool -P httpd_can_network_connect_db 1
    ```
    This ensures that the setting persists after a reboot.
   
This allows the web server to connect to remote databases.

## 10. **Commands to View Database from Command Line**
To view the contents of the database from the command line, log into the MySQL/MariaDB server and run the following commands:

Log into MySQL:

```
mysql -u ecomuser -p -h database-server-ip
```
Show the available databases:

```
SHOW DATABASES;
```
Use the ecomdb database:

```
USE ecomdb;
```
Show the tables in the ecomdb database:

```
SHOW TABLES;
```
View the data in the products table:

```
SELECT * FROM products;
```
By following the steps in this guide, you should be able to successfully deploy and configure the e-commerce application on CentOS, even in a multi-node environment. If any issues arise, refer to the troubleshooting section and try the solutions provided.
