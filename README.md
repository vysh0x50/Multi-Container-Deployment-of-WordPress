# Multi-Container-Deployment-of-WordPress

![Untitled Diagram](https://user-images.githubusercontent.com/65948438/162495574-7652e457-f7a7-43fc-bd7d-4e9921e60664.png)

## Introduction

WordPress is now used by over half of the top one million websites on the internet (Content Management System). WordPress is extremely user-friendly, especially for non-technical users, making it a popular CMS choice. Installing a LAMP (Linux, Apache, MySQL, and PHP) or LEMP (Linux, Nginx, MySQL, and PHP) stack to run WordPress is often time-consuming. You may ease the process of setting up your preferred stack and installing WordPress by using tools like Docker and Docker Compose.

You'll learn how to set up a multi-container WordPress installation in this article. A MySQL database, a Nginx web server, and WordPress itself will be included in your containers. You can additionally safeguard your installation by using Let's Encrypt to get TLS/SSL certificates for the domain you want to use with your site.



## Prerequisites

 - Docker installed on your server.
 - Docker Compose installed on your server.
 - A registered domain name. Here I will use blog.vyshnavlalp.ml throughout.
 - Domain's A record must be point to your server.
 
## Step 1 - Setting up project directory and Nginx configuration

Create a project directory called "wp-multicontainer" for your WordPress installation and navigate to it:
```sh
mkdir wp-multicontainer && cd wp-multicontainer
 ```
Next, make a directory for the Nginx configuration file and place the below conf to it. 

In this file, we'll add a server block with directives for our server name and document root, as well as location blocks to direct the Certbot client's certificate, PHP processing, and static asset requests.

```sh
server {
        listen 80;
        listen [::]:80;

        server_name blog.vyshnavlalp.ml;

        index index.php index.html index.htm;

        root /var/www/html;

        location ~ /.well-known/acme-challenge {
                allow all;
                root /var/www/html;
        }

        location / {
                try_files $uri $uri/ /index.php$is_args$args;
        }

        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_pass wordpress:9000;
                fastcgi_index index.php;
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_param PATH_INFO $fastcgi_path_info;
        }

        location ~ /\.ht {
                deny all;
        }
        
        location = /favicon.ico { 
                log_not_found off; access_log off; 
        }
        location = /robots.txt { 
                log_not_found off; access_log off; allow all; 
        }
        location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
                expires max;
                log_not_found off;
        }
}
```
For PHP requests, Nginx requires an independent PHP processor: These requests will be handled in our case by the php-fpm processor that comes with the php:fpm image. These requests are forwarded to our WordPress container.

## Step 2: Define Services using Docker Compose 

Create a docker-compose.yml file and add the below configuration to the file. The service definitions for your setup will be in the docker-compose.yml file and service definitions define how each container will run.

```sh
version: '3'
  
services:
  database:
    image: mysql:5.7
    container_name: database
    networks:
      - wpgrid
    volumes:
      - db-data:/var/lib/mysql/
    environment:
      MYSQL_ROOT_PASSWORD: <mysql root password>
      MYSQL_DATABASE: <database name>
      MYSQL_USER: <database user>
      MYSQL_PASSWORD:<database user password>
```
Here we are using mysql5.7 image with container name 'database' in the network 'wpgrid' which we will define at the bottom of file and the required environment variables.

Let's define our WordPress application service
```sh
  wordpress:
    image: wordpress:5.9.3-php7.4-fpm-alpine
    container_name: wordpress
    networks:
      - wpgrid
    volumes:
      - wp-data:/var/www/html/
    environment:
      WORDPRESS_DB_HOST: database
      WORDPRESS_DB_NAME: <database name>
      WORDPRESS_DB_USER: <database user>
      WORDPRESS_DB_PASSWORD: <database user password>
    depends_on:
      - database
```
For this setup, we are using the '5.9.3-php7.4-fpm-alpine' Wordpress image. This image ensures that our application has the php-fpm processor that Nginx needs to process PHP requests. This is also an alpine image, derived from the Alpine Linux project, which will assist reduce the size of our entire image. We’re also adding the wordpress container to the wpgrid network. Also by defining 'depends_on' wordpress container starting after the database container.

Next add the webserver Nginx service
```sh
  webserver:
    image: nginx:1.21.6-alpine
    container_name: webserver
    networks:
      - wpgrid
    volumes:
      - ./nginx/:/etc/nginx/conf.d/
      - ./logs/nginx:/var/log/nginx/
      - wp-data:/var/www/html/
      - certbot:/etc/letsencrypt/
    ports:
      - "80:80"
    depends_on:
      - wordpress
```
We’re using an alpine image — the 1.21.6-alpine Nginx image. This service definition also includes 
  - volumes: We're specifying both named volumes and bind mounts here:
      - ./nginx/:/etc/nginx/conf.d: This will bind mount the Nginx configuration directory on the host to the relevant directory on the container,               ensuring that any changes we make to files on the host will be reflected in the container.
      - ./logs/nginx:/var/log/nginx/ : This will bind mount the Nginx log directory on the host to the log directory on the container.
      - wp-data:/var/www/html/: This will mount our WordPress application code to the /var/www/html/ directory
      - certbot:/etc/letsencrypt/: This will mount the relevant Let’s Encrypt certificates and keys for our domain to the appropriate directory on the           container.
  - ports: This exposes port 80 to enable the configuration options we defined in our nginx.conf file.

Finally, add certbot service defenition.
```sh
  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot:/etc/letsencrypt/
      - wp-data:/var/www/html/
    command: certonly --webroot --webroot-path=/var/www/html/ --email <your email-id> --agree-tos --no-eff-email --staging -d blog.vyshnavlalp.ml
    depends_on:
      - webserver
```
This definition tells Compose to pull the certbot/certbot image from Docker Hub. It also uses volumes to share resources with the Nginx container, including the domain certificates and key in certbot-etc and the application code in wordpress.
      
We’ve also included a command option that specifies a subcommand to run with the container’s default certbot command. The certonly subcommand will obtain a certificate with the following options:
  - --webroot: This tells Certbot to use the webroot plugin to place files in the webroot folder for authentication.
  - --webroot-path: This specifies the path of the webroot directory.
  - --email: Your preferred email for registration and recovery.
  - --agree-tos: This specifies that you agree to ACME’s Subscriber Agreement.
  - --no-eff-email: This tells Certbot that you do not wish to share your email with the Electronic Frontier Foundation (EFF). Feel free to omit this if       you would prefer.
  - --staging: This tells Certbot that you would like to use Let’s Encrypt’s staging environment to obtain test certificates. Using this option allows         you to test your configuration options and avoid possible domain request limits.
 
Now add your network and volume definitions:
```sh
networks:
  wpgrid:

volumes:
  db-data:
  wp-data:
  certbot:
```

When Docker creates a volume, the contents are stored in a Docker-managed directory on the host filesystem under /var/lib/docker/volumes/. The contents of each volume are then mounted to any container that utilises the volume from this directory.

The user-defined bridge network wpgrid enables communication between our containers since they are on the same Docker daemon host. And it opens all ports between containers on the same bridge network without exposing any ports to the outside world, this simplifies traffic and communication within the application. As a result, our database, wordpress, and webserver containers can communicate with one another, and we only need to expose port 80 for application front-end access.

This is what the completed docker-compose.yml file will look like:
```sh
version: '3'
  
services:
  database:
    image: mysql:5.7
    container_name: database
    networks:
      - wpgrid
    volumes:
      - db-data:/var/lib/mysql/
    environment:
      MYSQL_ROOT_PASSWORD: <mysql root password>
      MYSQL_DATABASE: <database name>
      MYSQL_USER: <database user>
      MYSQL_PASSWORD: <database user password>

  wordpress:
    image: wordpress:5.9.3-php7.4-fpm-alpine
    container_name: wordpress
    networks:
      - wpgrid
    volumes:
      - wp-data:/var/www/html/
    environment:
      WORDPRESS_DB_HOST: database
      WORDPRESS_DB_NAME: <database name>
      WORDPRESS_DB_USER: <database user>
      WORDPRESS_DB_PASSWORD: <database user password>
    depends_on:
      - database
  
  webserver:
    image: nginx:1.21.6-alpine
    container_name: webserver
    networks:
      - wpgrid
    volumes:
      - ./nginx/:/etc/nginx/conf.d/
      - ./logs/nginx:/var/log/nginx/
      - wp-data:/var/www/html/
      - certbot:/etc/letsencrypt/
    ports:
      - "80:80"
    depends_on:
      - wordpress
      
  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot:/etc/letsencrypt/
      - wp-data:/var/www/html/
    command: certonly --webroot --webroot-path=/var/www/html/ --email <your email-id> --agree-tos --no-eff-email --staging -d blog.vyshnavlalp.ml
    depends_on:
      - webserver  

networks:
  wpgrid:

volumes:
  db-data:
  wp-data:
  certbot:
```

## Step 3 — Obtaining SSL Certificates and Credentials

We can start our containers with the docker-compose up command, which will create and run our containers in the order we have specified. If our domain requests are successful, the right certificates will place in the /etc/letsencrypt/live folder on the webserver container. Using docker-compose ps, check the status of your services:

If everything was successful, your database, wordpress, and webserver services will be Up and the certbot container will have exited with a 0 status message.

You can now check that your certificates have been mounted to the webserver container with docker-compose exec:
```sh
[ec2-user@ip-172-31-37-71 wp-multicontainer]$ docker-compose exec webserver ls -l /etc/letsencrypt/live/
total 4
-rw-r--r--    1 root     root           740 Apr  8 05:52 README
drwxr-xr-x    2 root     root            93 Apr  8 06:13 blog.vyshnavlalp.ml
```
So the request will be successful, you can edit the certbot service definition to remove the --staging flag and replcae it with --force-renewal flag, which will tell Certbot that you want to request a new certificate with the same domains as an existing certificate. Then you may run below command which will recreate the certbot container. Also include the --no-deps option to tell Compose that it can skip starting the webserver service, since it is already running:

```sh
[ec2-user@ip-172-31-37-71 wp-multicontainer]$ docker-compose up --force-recreate --no-deps certbot
```

After you've installed your certificates, you can alter your Nginx settings to include SSL.

## Step 4 — Modifying the Web Server Configuration and Service Definition

Adding an HTTP redirect to HTTPS, defining our SSL certificate and key locations, and adding security settings and headers are all part of enabling SSL in our Nginx configuration. Since you're going to recreate the webserver, stop the webserver container now.

```sh
[ec2-user@ip-172-31-37-71 wp-multicontainer]$ docker-compose stop webserver
Stopping webserver ... done
```
Before we modify the configuration file itself, let’s first get the recommended Nginx security parameters from Certbot. Create a file called 
options-ssl-nginx.conf, under 'nginx' directory and add the below into it.

```sh
# This file contains important security parameters. If you modify this file
# manually, Certbot will be unable to automatically provide future security
# updates. Instead, Certbot will print and log an error message with a path to
# the up-to-date file that you will need to refer to when manually updating
# this file.

ssl_session_cache shared:le_nginx_SSL:10m;
ssl_session_timeout 1440m;
ssl_session_tickets off;

ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers off;

ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384";
```
Next, remove the Nginx configuration file you created earlier and create another version of the file. Refer below:

```sh
server {
        listen 80;
        listen [::]:80;

        server_name blog.vyshnavlalp.ml;

        location ~ /.well-known/acme-challenge {
                allow all;
                root /var/www/html;
        }

        location / {
                rewrite ^ https://$host$request_uri? permanent;
        }
}

server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name blog.vyshnavlalp.ml;

        index index.php index.html index.htm;

        root /var/www/html;

        access_log /var/log/nginx/blog.vyshnavlalp.ml-access.log;
        error_log /var/log/nginx/blog.vyshnavlalp.ml-error.log;

        server_tokens off;

        ssl_certificate /etc/letsencrypt/live/blog.vyshnavlalp.ml/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/blog.vyshnavlalp.ml/privkey.pem;

        include /etc/nginx/conf.d/options-ssl-nginx.conf;

        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Referrer-Policy "no-referrer-when-downgrade" always;
        add_header Content-Security-Policy "default-src * data: 'unsafe-eval' 'unsafe-inline'" always;
        # add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
        # enable strict transport security only if you understand the implications

        location / {
                try_files $uri $uri/ /index.php$is_args$args;
        }

        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_pass wordpress:9000;
                fastcgi_index index.php;
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_param PATH_INFO $fastcgi_path_info;
        }

        location ~ /\.ht {
                deny all;
        }
        
        location = /favicon.ico { 
                log_not_found off; access_log off; 
        }
        location = /robots.txt { 
                log_not_found off; access_log off; allow all; 
        }
        location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
                expires max;
                log_not_found off;
        }
}
```

Before recreating the webserver service, you will need to add a 443 port mapping to your webserver service definition. Open your docker-compose.yml file and add "443:443" under port section of webserver service.

When done, the docker-compose.yml file should look like this:

```sh


version: '3'
  
services:
  database:
    image: mysql:5.7
    container_name: database
    networks:
      - wpgrid
    volumes:
      - db-data:/var/lib/mysql/
    environment:
      MYSQL_ROOT_PASSWORD: <mysql root password>
      MYSQL_DATABASE: <database name>
      MYSQL_USER: <database user>
      MYSQL_PASSWORD: <database password>

  wordpress:
    image: wordpress:5.9.3-php7.4-fpm-alpine
    container_name: wordpress
    networks:
      - wpgrid
    volumes:
      - wp-data:/var/www/html/
    environment:
      WORDPRESS_DB_HOST: database
      WORDPRESS_DB_NAME: <database name>
      WORDPRESS_DB_USER: <database user>
      WORDPRESS_DB_PASSWORD: <database user password>
    depends_on:
      - database

  webserver:
    image: nginx:1.21.6-alpine
    container_name: webserver
    networks:
      - wpgrid
    volumes:
      - ./nginx/:/etc/nginx/conf.d/
      - ./logs/nginx:/var/log/nginx/
      - wp-data:/var/www/html/
      - certbot:/etc/letsencrypt/
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - wordpress

  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot:/etc/letsencrypt/
      - wp-data:/var/www/html/
    command: certonly --webroot --webroot-path=/var/www/html/ --email <your email-id> --agree-tos --no-eff-email --force-renewal -d blog.vyshnavlalp.ml
    depends_on:
      - webserver

networks:
  wpgrid:

volumes:
  db-data:
  wp-data:
  certbot:
```

Now recreate the webserver service and can check our services after that.
```sh
[ec2-user@ip-172-31-37-71 wp-multicontainer]$ docker-compose up -d --force-recreate --no-deps webserver
Recreating webserver ... done

[ec2-user@ip-172-31-37-71 wp-multicontainer]$ docker-compose ps
  Name                 Command               State                                    Ports                                 
----------------------------------------------------------------------------------------------------------------------------
certbot     certbot certonly --webroot ...   Exit 0                                                                         
database    docker-entrypoint.sh mysqld      Up       3306/tcp, 33060/tcp                                                   
webserver   /docker-entrypoint.sh ngin ...   Up       0.0.0.0:443->443/tcp,:::443->443/tcp, 0.0.0.0:80->80/tcp,:::80->80/tcp
wordpress   docker-entrypoint.sh php-fpm     Up       9000/tcp   
```
Now that your containers are up and running, you can finish your WordPress installation via the web interface. Load your domain in web browser and complete the wordpress installation. I'm attaching my completed wordpress site setup.

![site1](https://user-images.githubusercontent.com/65948438/162490329-9e3fc8e5-7659-4055-a73e-82d4cc08c857.png)

If you learned something from my blog, please remember to share it ![image](https://user-images.githubusercontent.com/65948438/162490502-966a5d47-841b-4731-9fc0-7a6b935d360d.png)
 and if you REALLY enjoyed it, please follow me!






