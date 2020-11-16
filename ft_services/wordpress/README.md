# Wordpress in alpine

- 먼저, [MetalLB](https://www.notion.so/MetalLB-6d06bb696fa540cbbe67678fb960063c), [MySQL](https://www.notion.so/MySQL-in-alpine-d02b8abcb9e14c9ba8fe7ad8c8cdecea)이 설치되어 있어야 한다.

<details>
  <summary>그 전에, Docker로 Local환경에서 잘 작동하는 지 확인해 보자.(필수사항 아님)</summary>
  
- Dockerfile 작성

    `vim Dockerfile`

    ```jsx
    FROM alpine:latest

    RUN apk update
    # phpmyadmin때와 마찬가지로, php-fpm7을 실행하기 위한 패키지이다.
    RUN apk add nginx \
            php7 \
            php7-bcmath \
            php7-ctype \
            php7-curl \
            php7-fpm \
            php7-gd \
            php7-iconv \
            php7-intl \
            php7-json \
            php7-mbstring \
            php7-mcrypt \
            php7-mysqlnd \
            php7-mysqli \
            php7-opcache \
            php7-openssl \
            php7-pdo \
            php7-pdo_mysql \
            php7-pdo_pgsql \
            php7-pdo_sqlite \
            php7-phar \
            php7-posix \
            php7-session \
            php7-soap \
            php7-xml \
            php7-zip

    # php-fpm7, nginx, 그리고 wordpress의 설치를 진행하는 entry.sh
    COPY ./entry.sh /tmp/entry.sh
    # nginx 의 설정파일이 담긴 곳.
    COPY ./default.conf /etc/nginx/conf.d/
    # wordpress 설정파일
    COPY ./wp-config.php /tmp/

    ENTRYPOINT ["sh", "/tmp/entry.sh"]
    ```

- entry.sh

    `vim entry.sh`

    ```jsx
    # wordpress .tar 파일 다운로드
    wget https://wordpress.org/latest.tar.gz
    # tar 파일 압축해제
    tar -xvf latest.tar.gz
    # tar 파일 삭제
    rm -rf latest.tar.gz
    # wordpress가 들어갈 폴더 미리 확보
    mkdir -p /var/www/html
    # wordpress 폴더 이동
    mv wordpress /var/www/html
    # wordpress 설정파일 복사
    cp /tmp/wp-config.php /var/www/html/wordpress

    # php 구문 해석을 위한 php-fpm7 실행
    /usr/sbin/php-fpm7
    # 웹서버 가동을 위한 nginx 실행
    nginx -g "pid /tmp/nginx.pid; daemon off;"
    ```

- wp-config.php 작성

    `vim wp-config.php`

    ```jsx
    <?php
    /** The name of the database for WordPress */
    define( 'DB_NAME', 'wordpress' );

    /** MySQL database username */
    define( 'DB_USER', 'modyhoon' );

    /** MySQL database password */
    define( 'DB_PASSWORD', '1q2w3e4r' );

    /** MySQL hostname */
    define( 'DB_HOST', 'mysql-service' );

    /** Database Charset to use in creating database tables. */
    define( 'DB_CHARSET', 'utf8' );

    /** The Database Collate type. Don't change this if in doubt. */
    define( 'DB_COLLATE', '' );

    define( 'AUTH_KEY',         'put your unique phrase here' );
    define( 'SECURE_AUTH_KEY',  'put your unique phrase here' );
    define( 'LOGGED_IN_KEY',    'put your unique phrase here' );
    define( 'NONCE_KEY',        'put your unique phrase here' );
    define( 'AUTH_SALT',        'put your unique phrase here' );
    define( 'SECURE_AUTH_SALT', 'put your unique phrase here' );
    define( 'LOGGED_IN_SALT',   'put your unique phrase here' );
    define( 'NONCE_SALT',       'put your unique phrase here' );

    $table_prefix = 'wp_';

    define( 'WP_DEBUG', false );

    if ( ! defined( 'ABSPATH' ) ) {
      define( 'ABSPATH', __DIR__ . '/' );
    }

    require_once ABSPATH . 'wp-settings.php';
    ```

    - 이 분은 DB_NAME, DB_USER, DB_PASSWORD, DB_HOST 부분을 자기 상황에 맞게 잘 수정해주어야 한다.
        - MySQL에서 user와 db 가 살아있고, host부분에는 mysql의 [Container IP를 확인](https://www.notion.so/Docker-Container-IP-b0896cbc224d4bef9afb91e07b06faaf)해서 집어 넣어야 한다.
        - ⚠️ Cluster IP가 아니다!! 이건 Docker 환경이다.
- default.conf 파일 작성
    - 자체 nginx이므로 당연히 설정파일도 작성해주어야 한다.

    `vim default.conf`

    ```jsx
    # This is a default site configuration which will simply return 404, preventing
    # chance access to any other virtualhost.

    server {
      listen 5050 default_server;
      listen [::]:5050 default_server;

      root /var/www/html/wordpress;

      location / {
        try_files $uri $uri/ = 404;
      }

      location ~ \.php$ {
        include fastcgi.conf;
        fastcgi_index index.php;
        fastcgi_pass 127.0.0.1:9000;
      }

      index index.php;
    }
    ```

- 실행을 편하게 하기 위한 Makefile 작성

    `vim Makefile`

    ```jsx
    IMG_NAME	=	my_wordpress
    PS_NAME		=	wordpress-ps
    PORT1		=	5050

    all	:	build run

    run	:
      docker run --name $(PS_NAME) -d -p $(PORT1):$(PORT1) $(IMG_NAME)

    runit	:
      docker run --name $(PS_NAME) -it -p $(PORT1):$(PORT1) $(IMG_NAME)

    exec:
      docker exec -it $$(docker ps -aq -f "name=$(PS_NAME)") sh

    build	:
      docker build -t $(IMG_NAME) .

    rm	:
      docker rm -f $$(docker ps -f "name=$(PS_NAME)" -aq)

    rmi	:
      docker rmi -f $(IMG_NAME)
    ```

- 이제, 실행해보자.
    - 그 전에, mysql 컨테이너는 반드시 살아있어야 한다!
    - `make`
    - 크롬 창 하나를 띄우고 다음 주소를 입력한다

        `localhost:5050`

        ![Wordpress%20in%20alpine%20463505e6944b4343b44394b6a491b2b2/Untitled.png](Wordpress%20in%20alpine%20463505e6944b4343b44394b6a491b2b2/Untitled.png)

        ![Wordpress%20in%20alpine%20463505e6944b4343b44394b6a491b2b2/Untitled%201.png](Wordpress%20in%20alpine%20463505e6944b4343b44394b6a491b2b2/Untitled%201.png)

    - 성공적으로 접근이 된 모습.
    
</details>

## Step 1 : Dockerfile 작성

- `vim Dockerfile`

    ```jsx
    FROM alpine:latest

    RUN apk update
    RUN apk add vim
    RUN apk add nginx \
            php7 \
            php7-bcmath \
            php7-ctype \
            php7-curl \
            php7-fpm \
            php7-gd \
            php7-iconv \
            php7-intl \
            php7-json \
            php7-mbstring \
            php7-mcrypt \
            php7-mysqlnd \
            php7-mysqli \
            php7-opcache \
            php7-openssl \
            php7-pdo \
            php7-pdo_mysql \
            php7-pdo_pgsql \
            php7-pdo_sqlite \
            php7-phar \
            php7-posix \
            php7-session \
    	php7-soap \
            php7-xml \
            php7-zip

    COPY ./entry.sh /tmp/entry.sh
    COPY ./default.conf /etc/nginx/conf.d/
    COPY ./wp-config.php /tmp

    ENTRYPOINT ["sh", "/tmp/entry.sh"]
    ```

## Step 2 : entry.sh 작성

- `vim entry.sh`

    ```jsx
    wget https://wordpress.org/latest.tar.gz
    tar -xvf latest.tar.gz
    rm -rf latest.tar.gz
    mkdir -p /var/www/html
    mv wordpress /var/www/html
    cp /tmp/wp-config.php /var/www/html/wordpress

    /usr/sbin/php-fpm7
    nginx -g "pid /tmp/nginx.pid; daemon off;"
    ```

## Step 3 : default.conf 작성

- `vim default.conf`

    ```jsx
    # This is a default site configuration which will simply return 404, preventing
    # chance access to any other virtualhost.

    server {
    	listen 5050 default_server;
    	listen [::]:5050 default_server;

    	root /var/www/html/wordpress;

    	location / {
    		try_files $uri $uri/ = 404;
    	}

    	location ~ \.php$ {
    		include fastcgi.conf;
    		fastcgi_index index.php;
    		fastcgi_pass 127.0.0.1:9000;
    	}

    	index index.php;
    }
    ```

## Step 4 : wp-config.php 작성

- `vim wp-config.php`

    ```jsx
    <?php
    /** The name of the database for WordPress */
    define( 'DB_NAME', 'wordpress' );

    /** MySQL database username */
    define( 'DB_USER', 'modyhoon' );

    /** MySQL database password */
    define( 'DB_PASSWORD', '1q2w3e4r' );

    /** MySQL hostname */
    define( 'DB_HOST', 'mysql-service' );

    /** Database Charset to use in creating database tables. */
    define( 'DB_CHARSET', 'utf8' );

    /** The Database Collate type. Don't change this if in doubt. */
    define( 'DB_COLLATE', '' );

    define( 'AUTH_KEY',         'put your unique phrase here' );
    define( 'SECURE_AUTH_KEY',  'put your unique phrase here' );
    define( 'LOGGED_IN_KEY',    'put your unique phrase here' );
    define( 'NONCE_KEY',        'put your unique phrase here' );
    define( 'AUTH_SALT',        'put your unique phrase here' );
    define( 'SECURE_AUTH_SALT', 'put your unique phrase here' );
    define( 'LOGGED_IN_SALT',   'put your unique phrase here' );
    define( 'NONCE_SALT',       'put your unique phrase here' );

    $table_prefix = 'wp_';

    define( 'WP_DEBUG', false );

    if ( ! defined( 'ABSPATH' ) ) {
    	define( 'ABSPATH', __DIR__ . '/' );
    }

    require_once ABSPATH . 'wp-settings.php';
    ```

## Step 5 : wordpress.yaml 작성

- `vim wordpress.yaml`

    ```jsx
    apiVersion: v1
    kind: Service
    metadata:
        name: wordpress-service
        labels:
          type: app
          service: wordpress
    spec:
        ports:
        - port: 5050
          protocol: TCP
          targetPort: 5050
        selector:
            type: app
            service: wordpress
        type: LoadBalancer

    ---

    apiVersion: apps/v1
    kind: Deployment
    metadata:
        name: wordpress-deployment
        labels:
            type: app
            service: wordpress
    spec:
        selector:
            matchLabels:
                type: app
                service: wordpress
        strategy:
            type: Recreate
        template:
            metadata:
                labels:
                    type: app
                    service: wordpress
            spec:
                containers:
                - image: wordpress-image
                  imagePullPolicy: Never
                  name: wordpress-container
                  ports:
                  - containerPort: 5050
                  livenessProbe:
                    httpGet:
                      port: 5050
                    initialDelaySeconds: 5
                    periodSeconds: 5
    ```

## Step 6 : 확인

- 우선, mysql Pod가 살아있어야 정상 작동한다. 이전 과제를 먼저 수행하자.
- `eval $(minikube docker-env)`
- `docker build -t . wordpress-image`
- `kubectl apply -f wordpress.yaml`
- `kubectl get svc`

    ![Wordpress%20in%20alpine%20463505e6944b4343b44394b6a491b2b2/Untitled%202.png](Wordpress%20in%20alpine%20463505e6944b4343b44394b6a491b2b2/Untitled%202.png)

    - metalLB에 의해 ExternalIP가 할당되었을 것이다. chrome을 띄워서 확인해 보자. `<EXTERNAL_IP>:5050`

    ![Wordpress%20in%20alpine%20463505e6944b4343b44394b6a491b2b2/Untitled%203.png](Wordpress%20in%20alpine%20463505e6944b4343b44394b6a491b2b2/Untitled%203.png)

    ![Wordpress%20in%20alpine%20463505e6944b4343b44394b6a491b2b2/Untitled%204.png](Wordpress%20in%20alpine%20463505e6944b4343b44394b6a491b2b2/Untitled%204.png)
