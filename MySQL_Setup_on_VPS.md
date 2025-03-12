## Setup MySql on VPS

### 1. install MySQL on VPS

Update packets
```bash
apt update && apt upgrade -y
```

Install MySQL
```bash
apt install mysql-server -y
```

Verify that MySQL is installed
```bash
systemctl status mysql
```

If MySQL is not active, start it :
```bash
systemctl start mysql
systemctl enable mysql
```

### 2. Secure MySQL (optional but recommended)

```bash
mysql_secure_installation
```

Answer the questions:
- Configure VALIDATE PASSWORD plugin (Yes, recommended)
- Change root password (Yes, recommended)
- Delete anonymous users (Yes)
- Disable remote root access (Yes, unless remote access is required)
- Delete test database (Yes)
- Reload privileges (Yes)

### 3. Create a database

Connect to MySQL :
```bash
mysql -u root -p
```

Once connected, create a new database:
```bash
CREATE DATABASE test_db;
```

Next, create a MySQL user and give him/her the permissions :
```bash
CREATE USER 'user'@'localhost' IDENTIFIED BY 'strongpassword';
GRANT ALL PRIVILEGES ON test_db.* TO 'user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

MySQL URI to Connect with projects:
```bash
mysql://user_name@localhost:3306/database_name #without password
mysql://user_name:password@localhost:3306/database_name #with password
```

### 4. Some key points :

#### 1. User privileges :

Avoid `GRANT ALL PRIVILEGES` unless necessary. Give priority to specific rights:
```bash
GRANT SELECT, INSERT, UPDATE, DELETE ON test_db.* TO 'user'@'localhost';
```

#### 2. Remote access :

If the user needs remote access, replace 'localhost' with '%' (and configure the firewall):
```bash
CREATE USER 'user'@'%' IDENTIFIED BY 'strongpassword';
```