## Setup PostgreSQL on VPS

### 1. Update existing packages
```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install PostgreSQL
```bash
sudo apt install postgresql postgresql-contrib -y
```

### 3. Check PostgreSQL service
```bash
sudo systemctl status postgresql
```

### 4. Access PostgreSQL
```bash
sudo -u postgres psql
```

### 5. Set administrator password (in PSQL shell)
```sql
ALTER ROLE postgres WITH PASSWORD 'your_secure_password';
\q
```

### 6. Modify authentication (optional)

Edit the pg_hba.conf file:
```bash
sudo nano /etc/postgresql/*/main/pg_hba.conf
```

Modify authentication methods (e.g. change “peer” to “md5” for local connections)

### 7. Configure remote access (optional)

Edit postgresql.conf :
```bash
sudo nano /etc/postgresql/*/main/postgresql.conf
```

Modify the line :
```conf
listen_addresses = '*'
```

### 8. Open firewall port
```bash
sudo ufw allow 5432/tcp
sudo ufw reload
```

### 9. Restart PostgreSQL
```bash
sudo systemctl restart postgresql
```

### Safety tips :
- Always use strong passwords
- Limit remote access to necessary IPs
- Create specific users rather than using 'postgres'
- Update PostgreSQL regularly
- Set up automatic backups

To create a new user and database :
```bash
sudo -u postgres createuser --interactive
sudo -u postgres createdb database_name
```