# MySQL with PHPMyAdmin on Docker

This stack will boot a mysql container and a phpmyadmin container

## Compose 

The `docker-compose.yml`:

```yaml
version: '3.6'

services:
  phpmyadmin:
    image: phpmyadmin:latest
    container_name: phpmyadmin
    restart: unless-stopped
    environment:
      - PMA_ARBITRARY=1
      - TZ=Africa/Johannesburg
    networks:
      - public
    ports:
      - 18080:80
    depends_on:
      - mysql-db

  mysql-db:
    image: mysql:8.0
    container_name: mysql-db
    command: --default-authentication-plugin=mysql_native_password --init-file=/data/application/init.sql
    restart: unless-stopped
    security_opt:
      - seccomp:unconfined
    environment:
      - MYSQL_ROOT_PASSWORD=rootpassword
      - MYSQL_USER=ruan
      - MYSQL_PASSWORD=password
      - MYSQL_DATABASE=my_db
    volumes:
      - ./data:/var/lib/mysql
      - ./init.sql:/data/application/init.sql
    networks:
      - public

networks:
  public:
    name: public
```

We have a `init.sql` seed file to seed some dummy data:

```sql
CREATE DATABASE IF NOT EXISTS my_db;
CREATE TABLE IF NOT EXISTS my_db.users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    age INT(3) NOT NULL
);
INSERT IGNORE INTO my_db.users (id, name, age) VALUES (1, 'ruan', 34);
INSERT IGNORE INTO my_db.users (id, name, age) VALUES (2, 'stefan', 32);
INSERT IGNORE INTO my_db.users (id, name, age) VALUES (3, 'james', 28);
```

## Boot

Boot the stack with docker-compose:

```bash
docker-compose up -d
```

You can access PhpMyAdmin on port `18080` and the mysql hostname will be `mysql-db`, the root password will be `rootpassword`.
