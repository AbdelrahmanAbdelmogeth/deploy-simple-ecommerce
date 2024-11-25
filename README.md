# deploy-simple-ecommHere's the markdown for your GitHub repository:

markdown
Copy code
# E-Commerce Application Deployment on CentOS

This is a guide to deploying a simple e-commerce application on CentOS systems, built for learning purposes. The deployment involves setting up a multi-node environment where one server hosts the database and the other serves the web application.

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
2. Deploy and Configure Database
The database for the e-commerce application is MariaDB. Follow these steps to install and configure it.

Install MariaDB Server
bash
Copy code
sudo yum install -y mariadb-server
sudo systemctl start mariadb
sudo systemctl enable mariadb
Configure MariaDB
Start the MariaDB service and configure it:

bash
Copy code
sudo vi /etc/my.cnf
sudo systemctl restart mariadb
Configure Firewall for Database
Open port 3306 (MySQL default port) in the firewall to allow connections to the database server:

bash
Copy code
sudo firewall-cmd --permanent --zone=public --add-port=3306/tcp
sudo firewall-cmd --reload
Configure Database and User
Login to MariaDB and create the necessary database and user for the application:

bash
Copy code
$ mysql
MariaDB > CREATE DATABASE ecomdb;
MariaDB > CREATE USER 'ecomuser'@'localhost' IDENTIFIED BY 'ecompassword';
MariaDB > GRANT ALL PRIVILEGES ON *.* TO 'ecomuser'@'localhost';
MariaDB > FLUSH PRIVILEGES;
For a multi-node setup, replace localhost with the IP address of the web server where PHP is running (e.g., 'ecomuser'@'web-server-ip').

3. Load Product Inventory Information to Database
Create a SQL script to populate the database with some initial product information.

Create the db-load-script.sql
bash
Copy code
cat > db-load-script.sql <<-EOF
USE ecomdb;
CREATE TABLE products (id mediumint(8) unsigned NOT NULL auto_increment, Name varchar(255) default NULL, Price varchar(255) default NULL, ImageUrl varchar(255) default NULL, PRIMARY KEY (id)) AUTO_INCREMENT=1;

INSERT INTO products (Name, Price, ImageUrl) VALUES ("Laptop", "100", "c-1.png"), ("Drone", "200", "c-2.png"), ("VR", "300", "c-3.png"), ("Tablet", "50", "c-5.png"), ("Watch", "90", "c-6.png"), ("Phone Covers", "20", "c-7.png"), ("Phone", "80", "c-8.png"), ("Laptop", "150", "c-4.png");

EOF
Run the SQL Script
bash
Copy code
sudo mysql < db-load-script.sql
4. Deploy and Configure Web Server
The web server will host the PHP application. We will use Apache HTTP Server (httpd) along with PHP and the MySQLnd extension.

Install Required Packages
Install the necessary packages for Apache, PHP, and MySQL support:

bash
Copy code
sudo yum install -y httpd php php-mysqlnd
Configure Firewall for HTTP
Allow HTTP traffic by opening port 80 on the firewall:

bash
Copy code
sudo firewall-cmd --permanent --zone=public --add-port=80/tcp
sudo firewall-cmd --reload
Configure Apache
Change the default DirectoryIndex to make index.php the default page:

bash
Copy code
sudo sed -i 's/index.html/index.php/g' /etc/httpd/conf/httpd.conf
Start Apache HTTP Server
Start and enable the Apache service to start on boot:

bash
Copy code
sudo systemctl start httpd
sudo systemctl enable httpd
5. Download the Code
Download the e-commerce application code using Git:

bash
Copy code
sudo yum install -y git
sudo git clone https://github.com/kodekloudhub/learning-app-ecommerce.git /var/www/html/
6. Create and Configure the .env File
In the root directory of the project, create a .env file to store the database connection information.

bash
Copy code
cat > /var/www/html/.env <<-EOF
DB_HOST=localhost
DB_USER=ecomuser
DB_PASSWORD=ecompassword
DB_NAME=ecomdb
EOF
For multi-node setups, modify the DB_HOST value in the .env file to the IP address of the database server:

bash
Copy code
DB_HOST=192.168.x.x  # Database server IP address
7. Update index.php to Use Environment Variables
Modify index.php to load the environment variables from the .env file and use them for the database connection:

php
Copy code
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
8. Test the Connection
Test the connection to the web application by running the following curl command:

bash
Copy code
curl http://localhost
9. Common Issues Faced
Issue: Permission Denied
When trying to connect to the MySQL database from the PHP web server, you might encounter the following error:

yaml
Copy code
Connection failed: Permission denied
Solution: SELinux Policies
This issue is related to SELinux policies, which prevent the web server from connecting to a remote database by default. To resolve this:

Check the current SELinux boolean for HTTPD using:

bash
Copy code
getsebool -a | grep httpd
If httpd_can_network_connect_db is set to Off, enable it:

bash
Copy code
setsebool -P httpd_can_network_connect_db 1
This allows the web server to connect to remote databases.

10. Commands to View Database from Command Line
To view the contents of the database from the command line, log into the MySQL/MariaDB server and run the following commands:

Log into MySQL:

bash
Copy code
mysql -u ecomuser -p -h 192.168.56.103
Show the available databases:

sql
Copy code
SHOW DATABASES;
Use the ecomdb database:

sql
Copy code
USE ecomdb;
Show the tables in the ecomdb database:

sql
Copy code
SHOW TABLES;
View the data in the products table:

sql
Copy code
SELECT * FROM products;
By following the steps in this guide, you should be able to successfully deploy and configure the e-commerce application on CentOS, even in a multi-node environment. If any issues arise, refer to the troubleshooting section and try the solutions provided.

vbnet
Copy code

This markdown can now be posted on your GitHub repository to guide others through the deployment process.


# E-Commerce Application Deployment Guide

This guide provides the steps to deploy a sample e-commerce application on CentOS systems. It covers the process of setting up both the database server and the web server in a multi-node configuration.

## Problem Faced
While trying to connect the PHP web server to the MySQL database server, we encountered the error:


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

## Deploy Pre-Requisites

### Install FirewallD
1. Install and enable FirewallD:
    ```bash
    sudo yum install -y firewalld
    sudo systemctl start firewalld
    sudo systemctl enable firewalld
    sudo systemctl status firewalld
    ```

## Deploy and Configure Database

### Install MariaDB
1. Install MariaDB:
    ```bash
    sudo yum install -y mariadb-server
    sudo systemctl start mariadb
    sudo systemctl enable mariadb
    ```

### Configure Firewall for Database
1. Allow MySQL port through the firewall:
    ```bash
    sudo firewall-cmd --permanent --zone=public --add-port=3306/tcp
    sudo firewall-cmd --reload
    ```

### Configure Database
1. Log into MariaDB:
    ```bash
    mysql
    ```
2. Create the database and user:
    ```sql
    MariaDB > CREATE DATABASE ecomdb;
    MariaDB > CREATE USER 'ecomuser'@'localhost' IDENTIFIED BY 'ecompassword';
    MariaDB > GRANT ALL PRIVILEGES ON *.* TO 'ecomuser'@'localhost';
    MariaDB > FLUSH PRIVILEGES;
    ```
   - In a multi-node setup, replace `'localhost'` with the IP address of the web server.

## Load Product Inventory Information to Database

1. Create the `db-load-script.sql` file:
    ```bash
    cat > db-load-script.sql <<-EOF
    USE ecomdb;
    CREATE TABLE products (id mediumint(8) unsigned NOT NULL auto_increment,Name varchar(255) default NULL,Price varchar(255) default NULL, ImageUrl varchar(255) default NULL,PRIMARY KEY (id)) AUTO_INCREMENT=1;
    
    INSERT INTO products (Name,Price,ImageUrl) VALUES ("Laptop","100","c-1.png"),("Drone","200","c-2.png"),("VR","300","c-3.png"),("Tablet","50","c-5.png"),("Watch","90","c-6.png"),("Phone Covers","20","c-7.png"),("Phone","80","c-8.png"),("Laptop","150","c-4.png");
    EOF
    ```

2. Run the SQL script:
    ```bash
    sudo mysql < db-load-script.sql
    ```

## Deploy and Configure Web Server

### Install Required Packages
1. Install Apache and PHP:
    ```bash
    sudo yum install -y httpd php php-mysqlnd
    sudo firewall-cmd --permanent --zone=public --add-port=80/tcp
    sudo firewall-cmd --reload
    ```

### Configure Apache (httpd)
1. Change the `DirectoryIndex` to make `index.php` the default:
    ```bash
    sudo sed -i 's/index.html/index.php/g' /etc/httpd/conf/httpd.conf
    ```

2. Start Apache:
    ```bash
    sudo systemctl start httpd
    sudo systemctl enable httpd
    ```

### Download the Code
1. Install Git and clone the project:
    ```bash
    sudo yum install -y git
    sudo git clone https://github.com/kodekloudhub/learning-app-ecommerce.git /var/www/html/
    ```

### Create and Configure the `.env` File

1. Create the `.env` file in the root of your project folder:
    ```bash
    cat > /var/www/html/.env <<-EOF
    DB_HOST=localhost
    DB_USER=ecomuser
    DB_PASSWORD=ecompassword
    DB_NAME=ecomdb
    EOF
    ```

2. Update the `index.php` file to load environment variables and use them for database connection:
    ```php
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

    - In a multi-node setup, use the database server IP in the `.env` file as `DB_HOST`.

## Test the Setup

To test the web server and database connection, run the following command:
```bash
curl http://localhost






