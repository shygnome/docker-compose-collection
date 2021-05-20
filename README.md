# docker-compose-collection

## Contents

- [Database](#database)
- [Messaging](#Messaging)
- [Dashboard](#dashboard)
- [File Hosting](#file-hosting)

### Database

- [MySQL](#MySQL)


#### MySQL
```yaml
services:
  db:
    image: mysql:latest
    container_name: mysql
    restart: always
    ports:
    - "3306:3306"
    environment:
    MYSQL_ROOT_PASSWORD: password
    MYSQL_DATABASE: mydb
    MYSQL_USER: admin
    MYSQL_PASSWORD: admin
    #   MYSQL_ALLOW_EMPTY_PASSWORD:
    #   MYSQL_RANDOM_ROOT_PASSWORD:
    #   MYSQL_ONETIME_PASSWORD:
    volumes:
    - mysql:/var/lib/mysql

  phpmyadmin:
    image: phpmyadmin/phpmyadmin:latest
    container_name: phpmyadmin
    ports:
    - "18080:80"
    restart: unless-stopped
    depends_on:
    - db
    networks:
    - mysql
    environment:
    PMA_HOST: db

networks:
    mysql:

volumes:
    mysql:
```

### Messaging

- [Kafka](#Kafka)

#### Kafka
```yaml

```

### Dashboard

- [CartoDB](#CartoDB)

#### CartoDB
```yaml
version: '3.3'
services:
  cartodb:
    image: sverhoeven/cartodb
    hostname: ${HOSTNAME}
    ports:
        - "80:80"
    restart: always
    volumes:
      - cartodb_pgdata:/var/lib/postgresql

volumes:
  cartodb_pgdata:
```

### File Hosting

- [Nextcloud](#nextcloud)
- [ownCloud](#owncloud)

#### Nextcloud
```yaml
version: '3' 

services:

  proxy:
    image: jwilder/nginx-proxy:alpine
    labels:
      - "com.github.jrcs.letsencrypt_nginx_proxy_companion.nginx_proxy=true"
    container_name: nextcloud-proxy
    networks:
      - nextcloud_network
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./proxy/conf.d:/etc/nginx/conf.d:rw
      - ./proxy/vhost.d:/etc/nginx/vhost.d:rw
      - ./proxy/html:/usr/share/nginx/html:rw
      - ./proxy/certs:/etc/nginx/certs:ro
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/tmp/docker.sock:ro
    restart: unless-stopped
  
  letsencrypt:
    image: jrcs/letsencrypt-nginx-proxy-companion
    container_name: nextcloud-letsencrypt
    depends_on:
      - proxy
    networks:
      - nextcloud_network
    volumes:
      - ./proxy/certs:/etc/nginx/certs:rw
      - ./proxy/vhost.d:/etc/nginx/vhost.d:rw
      - ./proxy/html:/usr/share/nginx/html:rw
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    restart: unless-stopped

  db:
    image: mariadb
    container_name: nextcloud-mariadb
    networks:
      - nextcloud_network
    volumes:
      - db:/var/lib/mysql
      - /etc/localtime:/etc/localtime:ro
    environment:
      - MYSQL_ROOT_PASSWORD=toor
      - MYSQL_PASSWORD=mysql
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
    restart: unless-stopped
  
  app:
    image: nextcloud:latest
    container_name: nextcloud-app
    networks:
      - nextcloud_network
    depends_on:
      - letsencrypt
      - proxy
      - db
    volumes:
      - nextcloud:/var/www/html
      - ./app/config:/var/www/html/config
      - ./app/custom_apps:/var/www/html/custom_apps
      - ./app/data:/var/www/html/data
      - ./app/themes:/var/www/html/themes
      - /etc/localtime:/etc/localtime:ro
    environment:
      - VIRTUAL_HOST=nextcloud.YOUR-DOMAIN
      - LETSENCRYPT_HOST=nextcloud.YOUR-DOMAIN
      - LETSENCRYPT_EMAIL=YOUR-EMAIL
    restart: unless-stopped

volumes:
  nextcloud:
  db:

networks:
  nextcloud_network:
```

#### ownCloud
