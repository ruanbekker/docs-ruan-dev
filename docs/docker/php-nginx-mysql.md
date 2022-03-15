# Nginx, PHP and MySQL Stack

This stack will boot a nginx, php and mysql stack with docker-compose.

## Source Code

The source code for the example can be found on my github repository:
- https://github.com/ruanbekker/docker-php-nginx-mysql

## Compose

The `docker-compose.yml` for our services:

```yaml
version: '3.8'

services:
  nginx:
    build:
      context: ./nginx
    container_name: nginx
    restart: unless-stopped
    volumes:
      - ./app/index.php:/var/www/index.php
    depends_on:
      - php-fpm
      - database
    ports:
      - 8080:80
    networks:
      - php-stack

  php-fpm:
    build:
      context: ./php-fpm
    container_name: php-fpm
    restart: unless-stopped
    volumes:
      - ./app/index.php:/var/www/index.php
    depends_on:
      - database
    networks:
      - php-stack

  phpmyadmin:
    image: phpmyadmin:latest
    container_name: phpmyadmin
    restart: unless-stopped
    environment:
      - PMA_ARBITRARY=1
      - TZ=Africa/Johannesburg
    ports:
      - 8081:80
    depends_on:
      - database
    networks:
      - php-stack

  database:
    image: mariadb:latest
    container_name: database
    restart: unless-stopped
    environment:
      - MYSQL_DATABASE=mydb
      - MYSQL_USER=myuser
      - MYSQL_PASSWORD=secret
      - MYSQL_ROOT_PASSWORD=docker
    volumes:
      - ./database/data.sql:/docker-entrypoint-initdb.d/data.sql
    networks:
      - php-stack

networks:
  php-stack:
    name: php-stack 
```

## Application Code

In our `app/index.php` which is where our application code resides:

```php
<?php
$value = "World";
$db = new PDO('mysql:host=database;dbname=mydb;charset=utf8mb4', 'myuser', 'secret');
$databaseTest = ($db->query('SELECT * FROM users'))->fetchAll(PDO::FETCH_OBJ);
?>

<html>
    <body>
        <h1>Hello, <?= $value ?>!</h1>
        <?php foreach($databaseTest as $row): ?>
            <p>Hello, <?= $row->name ?></p>
        <?php endforeach; ?>
    </body>
</html>
```

## Database 

In our `database/data.sql` file we have the seed file while will create the table and insert the data into our mysql container:

```sql
DROP TABLE IF EXISTS `users`;
CREATE TABLE `users` (
  `name` varchar(20) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=latin1;

LOCK TABLES `users` WRITE;
INSERT INTO `users` VALUES ('ruan'),('frank'),('james');
UNLOCK TABLES;
```

## Nginx

For our nginx container, we specified a build directory to our dockerfile, which is: `nginx/Dockerfile` and has:

```dockerfile
FROM nginx:alpine
ADD nginx.conf /etc/nginx/nginx.conf
ADD conf.d/default.conf /etc/nginx/conf.d/default.conf
```

As you can see we are adding nginx configs, the first one is `nginx/nginx.conf`:

```
user  nginx;
worker_processes  4;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log main;
    sendfile        on;
    keepalive_timeout  65;
    include /etc/nginx/conf.d/default.conf;
}
```

And our application config `nginx/conf.d/default.conf` which you can see upstreams to our php-fpm container:

```
upstream php-upstream {
    server php-fpm:9000;
}

server {

    listen 80 default_server;
    listen [::]:80 default_server ipv6only=on;

    server_name localhost;
    root /var/www;
    index index.php index.html index.htm;

    location / {
         try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {
        try_files $uri /index.php =404;
        fastcgi_pass php-upstream;
        fastcgi_index index.php;
        fastcgi_buffers 16 16k;
        fastcgi_buffer_size 32k;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        #fixes timeouts
        fastcgi_read_timeout 600;
        include fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }

    location /.well-known/acme-challenge/ {
        root /var/www/letsencrypt/;
        log_not_found off;
    }
}
```

## PHP FPM

From the compose we have a build reference to `php-fpm/Dockerfile` which installs the pdo_mysql package:

```dockerfile
FROM php:fpm-alpine
RUN docker-php-ext-install pdo_mysql
```

## Boot the Stack

Because our compose have build references, we first need to build our containers before they can run:

```bash
docker-compose -f docker-compose.yml build
```

Then we can run our containers:

```bash
docker-compose -f docker-compose.yml up
```

When you access the application on http://localhost:8080 you should be able to see the data being returned from mysql via nginx and php. PHPMyAdmin is accessible via http://localhost:8081
