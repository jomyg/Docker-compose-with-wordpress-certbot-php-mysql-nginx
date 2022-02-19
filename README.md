# -Docker-compose-with-wordpress-certbot-php-mysql-nginx

[![Build](https://travis-ci.org/joemccann/dillinger.svg?branch=master)](https://travis-ci.org/joemccann/dillinger)

---

## Description

Running WordPress typically involves installing a LAMP (Linux, Apache, MySQL, and PHP) or LEMP (Linux, Nginx, MySQL, and PHP) stack, which can be time-consuming. However, by using tools like Docker and Docker Compose, you can simplify the process of setting up your preferred stack and installing WordPress. Instead of installing individual components by hand, you can use images, which standardize things like libraries, configuration files, and environment variables, and run these images in containers, isolated processes that run on a shared operating system. Additionally, by using Compose, you can coordinate multiple containers — for example, an application and database — to communicate with one another.

----
## Pre-Requests
- Need to install docker and docker-compose
-----

## Includes

> Containers we are creating (MySQL, WordPress+PHP-FPM, Nginx & certbot)
> Volumes (For MySQL, WordPress and certbot)
> Network 

### Docker installation 

```sh
yum install docker -y after docker installation, please start and enable it
```
### docker-compose installation

```sh
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/bin/docker-compose
sudo chmod +x /usr/bin/docker-compose
docker-compose version   
```

### Behind the code

> Now we can create a file called "docker-compose.yml" as a staging setup with pot 80 only
```sh
version: '3'

services:
  db:
    image: mysql:8.0
    container_name: db
    restart: unless-stopped
    env_file: .env
    environment:
      - MYSQL_DATABASE=wordpress
    volumes: 
      - dbdata:/var/lib/mysql
    command: '--default-authentication-plugin=mysql_native_password'
    networks:
      - app-network

  wordpress:
    depends_on: 
      - db
    image: wordpress:5.1.1-fpm-alpine
    container_name: wordpress
    restart: unless-stopped
    env_file: .env
    environment:
      - WORDPRESS_DB_HOST=db:3306
      - WORDPRESS_DB_USER=$MYSQL_USER
      - WORDPRESS_DB_PASSWORD=$MYSQL_PASSWORD
      - WORDPRESS_DB_NAME=wordpress
    volumes:
      - wordpress:/var/www/html
    networks:
      - app-network

  webserver:
    depends_on:
      - wordpress
    image: nginx:1.15.12-alpine
    container_name: webserver
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - wordpress:/var/www/html
      - ./nginx-conf:/etc/nginx/conf.d
      - certbot-etc:/etc/letsencrypt
    networks:
      - app-network

  certbot:
    depends_on:
      - webserver
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot-etc:/etc/letsencrypt
      - wordpress:/var/www/html
    command: certonly --webroot --webroot-path=/var/www/html --email jomyg@gmail.com --agree-tos --no-eff-email --staging -d jomygeorge.xyz -d www.jomygeorge.xyz

volumes:
  certbot-etc:
  wordpress:
  dbdata:

networks:
  app-network:
    driver: bridge
```
The confidential values that we will set in this file include a password for our MySQL root user, and a username and password that WordPress will use to access the database.

Add the following variable names and values to the file. Remember to supply your own values here for each variable:
```sh
wordpress]# cat .env
MYSQL_ROOT_PASSWORD=root@password
MYSQL_USER=wpuser
MYSQL_PASSWORD=mysql@password
```
Now create a Dir called "nginx-conf" and create a file nginx.conf under our project Dir ### Below is staging state, so the i haven't mentioned 443
~ wordpress]# cat nginx-conf/nginx.conf
```sh
server {
        listen 80;
        listen [::]:80;

        server_name jomygeorge.xyz www.jomygeorge.xyz;

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
We can start our containers with the docker-compose up command, which will create and run our containers in the order we have specified. If our domain requests are successful, we will see the correct exit status in our output and the right certificates mounted in the /etc/letsencrypt/live folder on the webserver container.

Create the containers with docker-compose up and the -d flag, which will run the db, wordpress, and webserver containers in the background:
This way we can genrate SSL cert when we run the compose file as the certbot is added

```
docker-compose up -d
```
Output
wordpress]#     docker-compose ps
NAME        COMMAND      SERVICE             STATUS      PORTS
certbot     certbot certonly --webroot ...   Exit 0                    ### After SSL creation it will get down
db          docker-entrypoint.sh --def ...   Up       3306/tcp, 33060/tcp
webserver   nginx -g daemon off;             Up       0.0.0.0:80->80/tcp 
wordpress   docker-entrypoint.sh php-fpm     Up       9000/tcp 
wordpress]#

> You can now check that your certificates have been mounted to the webserver container with docker-compose exec:
```
wordpress]# docker-compose exec webserver ls -la /etc/letsencrypt/live
total 4
drwx------    3 root     root            42 Feb 19 10:34 .
drwxr-xr-x    9 root     root           108 Feb 19 10:37 ..
-rw-r--r--    1 root     root           740 Feb 19 10:34 README
drwxr-xr-x    2 root     root            93 Feb 19 10:37 jomygeorge.xyz
```

> Now we can setup the 443 setup using the generated SSL , Inorder to do this, we need to change some rules in compose file and nginx conf file under our Dir. First we can chnage the compose file. Find the section of the file with the certbot service definition, and replace the --staging flag in the command option with the --force-renewal flag, which will tell Certbot that you want to request a new certificate with the same domains as an existing certificate. The certbot service definition will now look like this:

```
version: '3'

services:
  db:
    image: mysql:8.0
    container_name: db
    restart: unless-stopped
    env_file: .env
    environment:
      - MYSQL_DATABASE=wordpress
    volumes:
      - dbdata:/var/lib/mysql
    command: '--default-authentication-plugin=mysql_native_password'
    networks:
      - app-network

  wordpress:
    depends_on:
      - db
    image: wordpress:5.1.1-fpm-alpine
    container_name: wordpress
    restart: unless-stopped
    env_file: .env
    environment:
      - WORDPRESS_DB_HOST=db:3306
      - WORDPRESS_DB_USER=$MYSQL_USER
      - WORDPRESS_DB_PASSWORD=$MYSQL_PASSWORD
      - WORDPRESS_DB_NAME=wordpress
    volumes:
      - wordpress:/var/www/html
    networks:
      - app-network

  webserver:
    depends_on:
      - wordpress
    image: nginx:1.15.12-alpine
    container_name: webserver
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - wordpress:/var/www/html
      - ./nginx-conf:/etc/nginx/conf.d
      - certbot-etc:/etc/letsencrypt
    networks:
      - app-network

  certbot:
    depends_on:
      - webserver
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot-etc:/etc/letsencrypt
      - wordpress:/var/www/html
    command: certonly --webroot --webroot-path=/var/www/html --email jomyg@gmail.com --agree-tos --no-eff-email --force-renewal -d jomygeorge.xyz -d www.jomygeorge.xyz

volumes:
  certbot-etc:
  wordpress:
  dbdata:

networks:
  app-network:
    driver: bridge
 ```
> After above change in the compsose file. we can now run docker-compose up to recreate the certbot container. We will also include the --no-deps option to tell Compose that it can skip starting the webserver service, since it is already running: You will see output indicating that your certificate request was successful:
```
docker-compose up --force-recreate --no-deps certbot
```
> Now we can Modify the Web Server Configuration and Service Definition. Enabling SSL in our Nginx configuration will involve adding an HTTP redirect to HTTPS, specifying our SSL certificate and key locations. Since we are going to recreate the webserver service to include these additions, you can stop it now:
```sh
docker-compose stop webserver
```
> We need to add Nginx security parameters to Certbot using curl. This command will save these parameters in a file called options-ssl-nginx.conf, located in the nginx-conf dir.

```
curl -sSLo nginx-conf/options-ssl-nginx.conf https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf
```
-rw-r--r-- 1 root root 2219 Feb 19 10:40 nginx.conf
-rw-r--r-- 1 root root  721 Feb 19 10:38 options-ssl-nginx.conf
 nginx-conf]# cat options-ssl-nginx.conf

ssl_session_cache shared:le_nginx_SSL:10m;
ssl_session_timeout 1440m;
ssl_session_tickets off;

ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers off;

ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384";
 nginx-conf]#
 
 > Now we can chnage the conf file by including the SSL and 443 setup, nano nginx-conf/nginx.conf
```sh
server {
        listen 80;
        listen [::]:80;

        server_name jomygeorge.xyz www.jomygeorge.xyz;

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
        server_name jomygeorge.xyz www.jomygeorge.xyz;

        index index.php index.html index.htm;

        root /var/www/html;

        server_tokens off;

        ssl_certificate /etc/letsencrypt/live/jomygeorge.xyz/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/jomygeorge.xyz/privkey.pem;

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
> After the above conf change, add port - "443:443" on compose file under the nginx creation. I have give u an idea. Please see below

```
 webserver:
    depends_on:
      - wordpress
    image: nginx:1.15.12-alpine
    container_name: webserver
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"                               ####<--------------- There you go
    volumes:
      - wordpress:/var/www/html
      - ./nginx-conf:/etc/nginx/conf.d
      - certbot-etc:/etc/letsencrypt
    networks:
      - app-network
  ```
  
  > After all chnages in conf and compose file, we can recreate the webserver
  
```
  docker-compose up -d --force-recreate --no-deps webserver
```
nginx-conf]# docker container ls
CONTAINER ID   IMAGE                        COMMAND                  CREATED       STATUS       PORTS                                                                      NAMES
ba820b1c1e55   nginx:1.15.12-alpine         "nginx -g 'daemon of…"   3 hours ago   Up 3 hours   0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp   webserver
a172d98625bf   wordpress:5.1.1-fpm-alpine   "docker-entrypoint.s…"   3 hours ago   Up 3 hours   9000/tcp                                                                   wordpress
755210077c68   mysql:8.0                    "docker-entrypoint.s…"   3 hours ago   Up 3 hours   3306/tcp, 33060/tcp                                                        db

Below certbot is exited which is due to his work is completed. That why :)
nginx-conf]# docker container ls -a
CONTAINER ID   IMAGE                        COMMAND                  CREATED       STATUS                   PORTS                                                                      NAMES
ba820b1c1e55   nginx:1.15.12-alpine         "nginx -g 'daemon of…"   3 hours ago   Up 3 hours               0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp   webserver
253f937f468c   certbot/certbot              "certbot certonly --…"   3 hours ago   Exited (0) 3 hours ago                                                                              certbot
a172d98625bf   wordpress:5.1.1-fpm-alpine   "docker-entrypoint.s…"   3 hours ago   Up 3 hours               9000/tcp                                                                   wordpress
755210077c68   mysql:8.0                    "docker-entrypoint.s…"   3 hours ago   Up 3 hours               3306/tcp, 33060/tcp                                                        db
nginx-conf]#

> With our containers running, we can finish the installation through the WordPress web interface.



 ## Conclusion

Created the wordpress with php, mysql, certbot for SSL and Nginx  using docker compose


#### ⚙️ Connect with Me

<p align="center">
<a href="mailto:jomyambattil@gmail.com"><img src="https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white"/></a>
<a href="https://www.linkedin.com/in/jomygeorge11"><img src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white"/></a> 
<a href="https://www.instagram.com/therealjomy"><img src="https://img.shields.io/badge/Instagram-E4405F?style=for-the-badge&logo=instagram&logoColor=white"/></a><br />
