## Setup MariaDB on VPS

### 1. install MariaDB on VPS

Update packets
```bash
apt update && apt upgrade -y
```

Install MariaDB
```bash
apt install mariadb-server mariadb-client
```

Check that MariaDB is installed
```bash
systemctl status mariadb
```

If MariaDB is not active, start it :
```bash
systemctl start mariadb
systemctl enable mariadb
```

### 2. Secure MariaDB (Recommended)

```bash
mariadb-secure-installation
```

Answer the questions:
- Enter current password for root: Press Enter (no default password).
- Switch to unix_socket authentication: Y (additional security).
- Change the root password? : Y → Set a strong password.
- Remove anonymous users? : Y.
- Disallow root login remotely : Y (unless you need remote root access).
- Remove test database and access to it? : Y.
- Reload privilege tables now? : Y.

### 3. Create a database

Connect to MariaDB :
```bash
mariadb -u root -p
```

Once connected, create a new database:
```bash
CREATE DATABASE test_db;
```

Next, create a MariaDB user and give him/her the permissions :
```bash
CREATE USER 'user'@'localhost' IDENTIFIED BY 'strongpassword';
GRANT ALL PRIVILEGES ON test_db.* TO 'user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

MariaDB URI to Connect with projects:
```bash
mariadb://user_name@localhost:3306/database_name #without password
mariadb://user_name:password@localhost:3306/database_name #with password
```

### 4. Final check

Test the connection with the new user:
```bash
mariadb -u user -p # Enter password
```

Check databases:
```bash
SHOW DATABASES;
```

### 5. Some key points :

#### 1. Root authentication :

If you enabled Unix socket authentication when installing mariadb-secure, you can log in without a password, provided their system username matches an authorized MariaDB user via:
```bash
sudo mariadb -u root # without “-p”
```

#### 2. User privileges :

Avoid `GRANT ALL PRIVILEGES` unless necessary. Give priority to specific rights:
```bash
GRANT SELECT, INSERT, UPDATE, DELETE ON test_db.* TO 'user'@'localhost';
```

#### 3. Remote access :

If the user needs remote access, replace 'localhost' with '%' (and configure the firewall):
```bash
CREATE USER 'user'@'%' IDENTIFIED BY 'strongpassword';
```
