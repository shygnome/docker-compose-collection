# docker-compose-collection

## Contents

- [Database](#database)
- [Messaging](#Messaging)
- [Dashboard](#dashboard)

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


