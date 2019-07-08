创建 应用目录
-------------

```
[root@VM_16_3_centos /]# mkdir -p /data/dockerapp/testapp
[root@VM_16_3_centos /]# cd /data/dockerapp/testapp
[root@VM_16_3_centos testapp]# mkdir php
[root@VM_16_3_centos testapp]# mkdir www
[root@VM_16_3_centos testapp]# mkdir -p /nginx/conf.d
```


**NOTES:**
- `php`目录用于后续的构建PHP镜像
- `www`目录为PHP代码存放目录
- `/nginx/conf.d`目录为 nginx 配置文件的存储目录


docker运行mysql
---------------


```
docker run --name test-mysql -p 3307:3306 -d \
-e MYSQL_ROOT_PASSWORD=123456 \
mysql
```


docker运行PHP
-------------

### 创建Dockerfile

1. 在PHP目录下创建Dockerfile用于构建新的PHP镜像

    ```
    [root@VM_16_3_centos testapp]# cd php/
    [root@VM_16_3_centos php]# vim Dockerfile
    ```

2. 编写用于构建PHP镜像的Dockerfile：

    ```
    FROM php:7.3-fpm

    # Install system packages for PHP extensions recommended for Yii 2.0 Framework
    ENV DEBIAN_FRONTEND=noninteractive
    RUN apt-get update && \
        apt-get -y install \
            gnupg2 && \
        apt-key update && \
        apt-get update && \
        apt-get -y install \
                g++ \
                git \
                curl \
                imagemagick \
                libcurl3-dev \
                libicu-dev \
                libfreetype6-dev \
                libjpeg-dev \
                libjpeg62-turbo-dev \
                libmagickwand-dev \
                libpq-dev \
                libpng-dev \
                libxml2-dev \
                libzip-dev \
                zlib1g-dev \
                mysql-client \
                openssh-client \
                nano \
                unzip \
                libcurl4-openssl-dev \
                libssl-dev \
            --no-install-recommends && \
            apt-get clean && \
            rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

    # Install PHP extensions required for Yii 2.0 Framework
    RUN docker-php-ext-configure gd \
            --with-freetype-dir=/usr/include/ \
            --with-png-dir=/usr/include/ \
            --with-jpeg-dir=/usr/include/ && \
        docker-php-ext-configure bcmath && \
        docker-php-ext-install \
            soap \
            zip \
            curl \
            bcmath \
            exif \
            gd \
            iconv \
            intl \
            mbstring \
            opcache \
            pdo_mysql \
            pdo_pgsql

    # Install PECL extensions
    # see http://stackoverflow.com/a/8154466/291573) for usage of `printf`
    RUN printf "\n" | pecl install \
            imagick \
            mongodb && \
        docker-php-ext-enable \
            imagick \
            mongodb
    ```

3. 构建PHP镜像

    ```
    [root@VM_16_3_centos php]# docker build -t php:v73fpmyii2 .
    ```

    构建成功后用`docker image ls`命令可查看到该镜像：

    ```
    [root@VM_16_3_centos testapp]# docker image ls
    REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
    php                 v73fpmyii2          71556294ee27        About an hour ago   743MB
    php                 7.3-fpm             6e2d39230895        4 days ago          371MB
    ```

4. 运行PHP容器

    ```
    docker run --name test-php -d \
    -v /data/dockerapp/testapp/www:/var/www/html \
    --link test-mysql:mysql \
    php:v73fpmyii2
    ```

    **NOTES:**
    - `/var/www/html`为`php`容器中的脚本运行目录
    - `link test-mysql:mysql`用于`php`容器与`mysql`容器互通，如在`php`代码中连接`Pdo`示例代码：
       `'dsn' => 'mysql:host=mysql;dbname=test'`，`host`为`mysql`而不是`localhost`或者IP



docker运行nginx
---------------



1. 运行nginx容器

    ```
    docker run --name test-nginx -p 10080:80 -d \
    -v /data/dockerapp/testapp/www:/usr/share/nginx/html \
    -v /data/dockerapp/testapp/nginx/conf.d:/etc/nginx/conf.d \
    --link test-php:php \
    nginx
    ```

    **NOTES:**
    - `/usr/share/nginx/html`为`nginx`容器中的web目录
    - `/etc/nginx/conf.d`为`nginx`容器中`vhosts`的配置目录
    - `--link test-php:php`用于连通`php`容器，保证`nginx`容器与`php`容器互通，让`nginx`通过`php:9000`访问`php-fpm`

2. 设置yii2的nginx的配置

    ```
    [root@VM_16_3_centos testapp]# vim nginx/conf.d/testyii2.conf
    ```

    配置内容如下：

    ```
    server {
        charset utf-8;
        client_max_body_size 128M;

        listen       80;
        server_name  localhost;

        root        /usr/share/nginx/html/my_yii2_basic/web;
        index       index.php;

        location / {
            try_files $uri $uri/ /index.php$is_args$args;
        }

        location ~ ^/assets/.*\.php$ {
            deny all;
        }

        location ~ \.php$ {
            include        fastcgi_params;
            fastcgi_param  SCRIPT_FILENAME  /var/www/html/my_yii2_basic/web/$fastcgi_script_name;
            fastcgi_pass   php:9000;
            try_files $uri =404;
        }

        location ~* /\. {
            deny all;
        }
    }
    ```

    **NOTES:**
    - `/usr/share/nginx/html`和`/var/www/html`都是对应本应用中的`/www`目录。
    - `root /usr/share/nginx/html/my_yii2_basic/web;`这里是`nginx`容器中的对应目录。
    - `fastcgi_param  SCRIPT_FILENAME  /var/www/html/my_yii2_basic/web/$fastcgi_script_name;` 这里是`php`容器中对应的目录。


测试应用
--------

用`docker ps`命令可查看已经运行了的容器


```
[root@VM_16_3_centos testapp]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                               NAMES
26b36008f415        nginx               "nginx -g 'daemon of…"   2 hours ago         Up 11 minutes       0.0.0.0:10080->80/tcp               test-nginx
d049cf0b2a31        php:v73fpmyii2      "docker-php-entrypoi…"   2 hours ago         Up 2 hours          9000/tcp                            test-php
7a310abd1b99        mysql               "docker-entrypoint.s…"   5 days ago          Up 2 hours          33060/tcp, 0.0.0.0:3307->3306/tcp   test-mysql
```


在浏览器中访问`IP:10080`可看到yii的欢迎界面。over ~