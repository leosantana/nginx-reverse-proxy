# nginx-reverse-proxy
Esse container quando ativo detecta quando outros containers são iniciados e cria vhosts automaticamente.


## COMO USAR

1 - Informar email na seção de variáveis do letsencrypt no docker compose
```yml
    environment:
      DEFAULT_EMAIL: your@email.com
```

2 - Criar uma rede do docker onde o container do nginx vai estar conectado junto com os demais containers web
```ps
$ docker network create sites
```

3 - O container web a ser criado deve conter as variáveis **VIRTUAL_HOST** e **LETSENCRYPT_HOST** e a rede previamente criada como no exemplo abaixo:
```yml
version: '3'

services:
  db:
    image: mysql:8.0
    container_name: db
    restart: unless-stopped
    env_file: .env
    environment:
      - MYSQL_DATABASE=$MYSQL_DATABASE
    ports:
      - "3306:3306"
    volumes:
      - ./dbdata:/var/lib/mysql:delegated
    command: '--default-authentication-plugin=mysql_native_password'
    networks:
      - my-network

  wp:
    depends_on:
      - db
    image: wordpress:php8.0-apache
    container_name: wp
    restart: always
    env_file: .env
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: $MYSQL_USER
      WORDPRESS_DB_PASSWORD: $MYSQL_PASSWORD
      WORDPRESS_DB_NAME: $MYSQL_DATABASE
      WORDPRESS_CONFIG_EXTRA: |
          define('FS_METHOD', 'direct');

          define( 'AUTOSAVE_INTERVAL', 60*60*60*24*365 );
          define( 'EMPTY_TRASH_DAYS',  0 );
          define( 'WP_POST_REVISIONS', false );
          define( 'DISABLE_WP_CRON', true );

          define( 'WP_HOME', '$URL' );
          define( 'WP_SITEURL', '$URL' );

          define( 'WP_REDIS_HOST', 'redis' );
          define( 'WP_REDIS_PORT', 6379 );
          define( 'WP_REDIS_TIMEOUT', 1 );
          define( 'WP_REDIS_READ_TIMEOUT', 1 );

          define( 'WP_REDIS_DATABASE', 0 );
          define( 'AUTOMATIC_UPDATER_DISABLED', true);

          @ini_set( 'upload_max_filesize' , '5G' );
          @ini_set( 'post_max_size', '5G');
          @ini_set( 'memory_limit', '256M' );
          @ini_set( 'max_execution_time', '300' );
          @ini_set( 'max_input_time', '300' );
      VIRTUAL_HOST: example.com.br,www.example.com.br
      LETSENCRYPT_HOST: example.com.br,www.example.com.br
    volumes:
      - ./../src:/var/www/html
      - ./php-conf/php.ini:/usr/local/etc/php/php.ini
    networks:
      - my-network
      - sites

  redis:
    image: redis:6.2.5-alpine
    container_name: redis
    restart: unless-stopped
    volumes:
      - ./redis-conf:/usr/local/etc/redis/redis.conf
    command: 'redis-server /usr/local/etc/redis/redis.conf'
    networks:
      - my-network

networks:
  my-network:
    internal: true
  sites:
    external: true
```

4 - Inicie o container do nginx-reverse-proxy primeiro
```ps
$ cd nginx-reverse-proxy
$ docker-compose up -d
```
