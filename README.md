# docker-compose-collection

## Contents

- [Database](#database)
- [Messaging](#messaging)
- [Dashboard](#dashboard)
- [File Hosting](#file-hosting)
- [Object Storage](#object-storage)
- [Image Proxy](#image-proxy)

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

- [Kafka](#kafka)
- [RabbitMQ](#rabbitmq)

#### Kafka
Ref: https://dev.to/thegroo/one-to-run-them-all-1mg6
```yaml
version: '3.2'

services:
  # https://github.com/wurstmeister/zookeeper-docker
  zookeeper:
    container_name: zookeeper
    image: wurstmeister/zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports:
    - "2181:2181"

  # https://hub.docker.com/r/confluentinc/cp-kafka/
  kafka:
    container_name: kafka
    image: wurstmeister/kafka:2.12-2.2.1
    environment:
      ## the >- used below infers a value which is a string and properly 
      ## ignore the multiple lines resulting in one long string: 
      ## https://yaml.org/spec/1.2/spec.html
      KAFKA_ADVERTISED_LISTENERS: >- 
        LISTENER_DOCKER_INTERNAL://kafka:19092, 
        LISTENER_DOCKER_EXTERNAL://${DOCKER_HOST_IP:-kafka}:9092

      KAFKA_LISTENERS: >-
        LISTENER_DOCKER_INTERNAL://:19092,
        LISTENER_DOCKER_EXTERNAL://:9092

      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: >- 
        LISTENER_DOCKER_INTERNAL:PLAINTEXT,
        LISTENER_DOCKER_EXTERNAL:PLAINTEXT

      KAFKA_INTER_BROKER_LISTENER_NAME: LISTENER_DOCKER_INTERNAL
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_LOG4J_LOGGERS: >- 
        kafka.controller=INFO,
        kafka.producer.async.DefaultEventHandler=INFO,
        state.change.logger=INFO

    ports:
    - 9092:9092
    depends_on:
    - zookeeper
    volumes:
    - /var/run/docker.sock:/var/run/docker.sock
```

#### RabbitMQ
```yaml
version: "3.2"
services:
  rabbitmq:
    image: rabbitmq:3
    command: >
        /bin/bash -c "rabbitmq-plugins enable rabbitmq_mqtt;
                      rabbitmq-plugins enable rabbitmq_web_mqtt;
                      rabbitmq-plugins enable rabbitmq_management;
                      rabbitmq-plugins enable rabbitmq_amqp1_0;
                      rabbitmq-server"
    container_name: 'rabbitmq'
    ports:
        - 5672:5672
        - 15672:15672
        - 1883:1883
        - 15675:15675
    environment:
        RABBITMQ_DEFAULT_USER: ${RABBITMQ_DEFAULT_USER}
        RABBITMQ_DEFAULT_PASS: ${RABBITMQ_DEFAULT_PASS}
    volumes:
        - ./.docker/rabbitmq/data/:/var/lib/rabbitmq/
        - ./.docker/rabbitmq/log/:/var/log/rabbitmq
        - ./.docker/rabbitmq/etc/:/etc/rabbitmq/
    networks:
        - rabbitmq_go_net

networks:
  rabbitmq_go_net:
    driver: bridge
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
Ref: https://github.com/bitnami/bitnami-docker-owncloud.git
```yaml
version: '2'
services:
  mariadb:
    image: docker.io/bitnami/mariadb:10.3
    environment:
      - ALLOW_EMPTY_PASSWORD=yes
      - MARIADB_USER=bn_owncloud
      - MARIADB_DATABASE=bitnami_owncloud
    volumes:
      - 'mariadb_data:/bitnami/mariadb'
  owncloud:
    image: docker.io/bitnami/owncloud:10
    ports:
      - '11080:8080'
      - '11443:8443'
    environment:
      - OWNCLOUD_DATABASE_HOST=mariadb
      - OWNCLOUD_DATABASE_PORT_NUMBER=3306
      - OWNCLOUD_DATABASE_USER=bn_owncloud
      - OWNCLOUD_DATABASE_NAME=bitnami_owncloud
      - ALLOW_EMPTY_PASSWORD=yes
      # Host for accessing ownCloud
      # note: this setting will only be applied on the first run
      # ref: https://github.com/bitnami/bitnami-docker-owncloud#configuration
      - OWNCLOUD_HOST=localhost
    volumes:
      - 'owncloud_data:/bitnami/owncloud'
    depends_on:
      - mariadb
volumes:
  mariadb_data:
    driver: local
  owncloud_data:
    driver: local
```

### Object Storage

- [CloudServer]()
- [MinIO](#minio)

#### MinIO
Reference: https://docs.min.io/docs/deploy-minio-on-docker-compose.html

`docker-compose.yaml`

```yaml
version: '3.7'

# starts 4 docker containers running minio server instances.
# using nginx reverse proxy, load balancing, you can access
# it through port 9000.
services:
  minio1:
    image: minio/minio:RELEASE.2021-05-18T00-53-28Z
    volumes:
      - data1-1:/data1
      - data1-2:/data2
    expose:
      - "9000"
    environment:
      MINIO_ROOT_USER: minio
      MINIO_ROOT_PASSWORD: minio123
    command: server http://minio{1...4}/data{1...2}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  minio2:
    image: minio/minio:RELEASE.2021-05-18T00-53-28Z
    volumes:
      - data2-1:/data1
      - data2-2:/data2
    expose:
      - "9000"
    environment:
      MINIO_ROOT_USER: minio
      MINIO_ROOT_PASSWORD: minio123
    command: server http://minio{1...4}/data{1...2}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  minio3:
    image: minio/minio:RELEASE.2021-05-18T00-53-28Z
    volumes:
      - data3-1:/data1
      - data3-2:/data2
    expose:
      - "9000"
    environment:
      MINIO_ROOT_USER: minio
      MINIO_ROOT_PASSWORD: minio123
    command: server http://minio{1...4}/data{1...2}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  minio4:
    image: minio/minio:RELEASE.2021-05-18T00-53-28Z
    volumes:
      - data4-1:/data1
      - data4-2:/data2
    expose:
      - "9000"
    environment:
      MINIO_ROOT_USER: minio
      MINIO_ROOT_PASSWORD: minio123
    command: server http://minio{1...4}/data{1...2}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  nginx:
    image: nginx:1.19.2-alpine
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "9000:9000"
    depends_on:
      - minio1
      - minio2
      - minio3
      - minio4

## By default this config uses default local driver,
## For custom volumes replace with volume driver configuration.
volumes:
  data1-1:
  data1-2:
  data2-1:
  data2-2:
  data3-1:
  data3-2:
  data4-1:
  data4-2:
```
`nginx.conf`
```apacheconf
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  4096;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    keepalive_timeout  65;

    # include /etc/nginx/conf.d/*.conf;

    upstream minio {
        server minio1:9000;
        server minio2:9000;
        server minio3:9000;
        server minio4:9000;
    }

    server {
        listen       9000;
        listen  [::]:9000;
        server_name  localhost;

        # To allow special characters in headers
        ignore_invalid_headers off;
        # Allow any size file to be uploaded.
        # Set to a value such as 1000m; to restrict file size to a specific value
        client_max_body_size 0;
        # To disable buffering
        proxy_buffering off;

        location / {
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_connect_timeout 300;
            # Default is HTTP/1, keepalive is only enabled in HTTP/1.1
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            chunked_transfer_encoding off;

            proxy_pass http://minio;
        }
    }
}
```
