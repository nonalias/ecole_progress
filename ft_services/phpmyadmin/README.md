# phpMyAdmin in alpine

- 먼저, [nginx](https://www.notion.so/Nginx-and-SSH-SSL-protocol-in-alpine-13cf9068e2d24408bee140212919e855)와 [mysql](https://www.notion.so/MySQL-in-alpine-d02b8abcb9e14c9ba8fe7ad8c8cdecea)이 제대로 작동되고 있어야 한다.
- 시작하기에 앞서, Docker Container로 먼저 작동시켜보자.(필수사항 아님.)
    - 우선 앞서 진행했던 mysql이 작동되고 있어야 한다.
        - 그 이유는, phpMyAdmin은 mySQL을 편하게 관리할 수 있도록 해주는 tool 이기 때문이다.
        - 그러므로 당연히 조작할 DB가 작동되고 있어야 한다.
    - 먼저, Dockerfile을 작성해보자.
        - `vim Dockerfile`

            ```jsx
            FROM alpine:latest

            RUN apk update
            RUN apk add vim

            # nginx, php-fpm7에 대한 패키지 설치 나머지는 안깔면 phpmyadmin에서 깔라고 한다.
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

            # 5000번 포트로 진입하기 때문에 5000이다.
            EXPOSE 5000
            # 자체 nginx을 구동해야 하므로 default.conf파일을 복사해준다. 
            # entry.sh는 ENTRYPOINT에서 실행할 파일이다.
            COPY entry.sh /tmp/entry.sh
            COPY default.conf /etc/nginx/conf.d/

            # phpmyadmin이 담길 폴더를 미리 만들어준다.
            RUN mkdir -p /var/www/html

            # phpmyadmin을 다운받고, 압축을 풀고, 이름을 간단하게 바꾸고, /var/www/html으로 옮겨준다.
            RUN wget https://files.phpmyadmin.net/phpMyAdmin/5.0.2/phpMyAdmin-5.0.2-all-languages.tar.gz
            RUN tar -xvf phpMyAdmin-5.0.2-all-languages.tar.gz
            RUN rm -f phpMyAdmin-5.0.2-all-languages.tar.gz
            RUN mv phpMyAdmin-5.0.2-all-languages phpmyadmin
            RUN mv /phpmyadmin /var/www/html/

            # phpmyadmin 설정파일을 복붙해준다. 여기서 host의 정보를 수정한다.
            COPY config.inc.php /var/www/html/phpmyadmin

            ENTRYPOINT ["sh", "/tmp/entry.sh"]
            ```

    - nginx 설정파일인 default.conf를 만들어보자.
        - `vim default.conf`

            ```jsx
            # This is a default site configuration which will simply return 404, preventing
            # chance access to any other virtualhost.

            server {
            	listen 5000 default_server;
            	listen [::]:5000 default_server;

            	root /var/www/html/phpmyadmin;

            	location / {
            		try_files $uri $uri/ = 404;
            	}

            # php 문법을 해석하기 위한 구문이다. 
            # 이 구문이 있어야 *.php 파일을 해석할 수 있다.(cgi가 해석한다.)
            	location ~ \.php$ {
            		include fastcgi.conf;
            		fastcgi_index index.php;
            		fastcgi_pass 127.0.0.1:9000;
            	}

            	index index.php;
            }
            ```

    - phpmyadmin 설정파일인 config.inc.php를 작성해보자.
        - 여기서, 주의해야 할 점은 host를 나중에 mysql 컨테이너가 실행됬을 때 **그 아이피로 바꾸어주어야** 한다는 점이다.
        - `vim config.inc.php`

            ```jsx
            <?php
            declare(strict_types=1);

            $cfg['blowfish_secret'] = 'sibal_ft_services_fucking_blowfi'; /* YOU MUST FILL IN THIS FOR COOKIE AUTH! */
            $i = 0;
            $i++;
            $cfg['Servers'][$i]['auth_type'] = 'cookie';
            $cfg['Servers'][$i]['host'] = '172.17.0.3'; // **이 부분을 반드시 바꿔줘야 한다!!!**
            $cfg['Servers'][$i]['compress'] = false;
            $cfg['Servers'][$i]['AllowNoPassword'] = false;

            $cfg['UploadDir'] = '';
            $cfg['SaveDir'] = '';
            ```

    - 이제 필요한 명령어들을 실행하는 shell 파일을 작성해보자.
        - `vim entry.sh`

            ```jsx
            /usr/sbin/php-fpm7
            nginx -g 'pid /tmp/nginx.pid; daemon off;'
            ```

    - Docker를 편하게 빌드, 실행할 수 있게 해주는 Makefile을 만들어 보자. (명령어가 익숙하다면 할 필요 없음)
        - `vim Makefile`

            ```jsx
            IMG_NAME	=	my_phpmyadmin
            PS_NAME		=	phpmyadmin-ps
            PORT1		=	5000

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

    - 이제, mysql container를 실행해 보자.
        - `cd ../mysql/local_not_necessary`
        - `make build`
        - `make run`
    - 그리고 나서, mysql의 IP를 확인하자.
        - `docker ps`  → 이걸로 Container ID 4자리 를 확인,
        - `docker inspect <Container ID> | grep IPAddress`
        - 이 명령어를 실행하면 해당 IP가 나올 것이다.
        - 이제 그 IP를 외워서 config.inc.php에 있는 host에 대입시켜주자.
    - 대입이 완료되었다면, phpmyadmin의 container를 실행시켜줄 차례다.
        - `make`
    - 이제, 브라우저를 열고 다음과 같이 입력하자
        - `localhost:5000`
        - modyhoon:1q2w3e4r 를 입력해서 들어가지면 완료.
    - 헌데, 이 아이피가 계속 바뀌면 설정파일도 계속 바꿔주어야 할까?
        - 정답은 아니다. Docker Container 환경에서는 매번 컨테이너의 아이피가 랜덤으로 배정받기 때문에 건드려 줘야한다. but  --link 라는 방법도 있지만 선호되지 않음.
        - k8s에서는 서비스 이름만 넣으면 [저절로 그 서비스의 ClusterIP가 입력](https://www.notion.so/Kubernetes-DNS-a3b707350c324444878d918543a0e65d)된다. 따라서 같은 Cluster 내에 있으면 해당 IP로 접근할 수 있다.

## Step 1 : Dockerfile 작성

- 늘 그랬듯, 로컬 이미지를 생성하기 위하여 Dockerfile을 작성해주어야 한다.

    `vim Dockerfile`

    ```jsx
    FROM alpine:latest

    RUN apk update

    # php를 해석하기 위한 프로그램(php-rpm)을 위해 아래와같은 패키지를 설치해야한다.
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

    # Expose는 꼭 해줄 필요는 없으나, 명시를 위해 적어두는 것이 좋다.
    EXPOSE 5000
    # 실제로 phpmyadmin을 받아오고, php를 해석하기 위한 php-fpm을 실행하는 sh파일
    COPY entry.sh /tmp/entry.sh
    # 자체 nginx을 가져야 하므로, 당연히 nginx의 설정파일도 가지고 있어야한다.
    COPY default.conf /etc/nginx/conf.d/
    # phpmyadmin의 설정파일
    COPY config.inc.php /tmp/

    ENTRYPOINT ["sh", "/tmp/entry.sh"]
    ```

## Step 2 : entry.sh 작성하기

- 실제로 phpmyadmin을 설치하고, 연동하는 작업을 진행하는 entry.sh파일을 작성해준다.

    `vim entry.sh`

    ```jsx
    # phpmyadmin이 보관될 위치를 미리 확보해 놓는다.
    mkdir -p /var/www/html
    # phpmyadmin을 .tar파일 형태로 다운받는다.
    wget https://files.phpmyadmin.net/phpMyAdmin/5.0.2/phpMyAdmin-5.0.2-all-languages.tar.gz
    # .tar 파일의 압축을 해제한다.
    tar -xvf phpMyAdmin-5.0.2-all-languages.tar.gz
    # 압축 해제한 후 원본파일은 삭제한다.
    rm -f phpMyAdmin-5.0.2-all-languages.tar.gz
    # phpmyadmin 폴더를 알아보기 쉽게 이름을 수정한다.
    mv phpMyAdmin-5.0.2-all-languages phpmyadmin
    # nginx의 root를 /var/www/html으로 설정했기 때문에 하위폴더에 phpmyadmin을 넣어준다.
    mv /phpmyadmin/ /var/www/html/
    # 미리 만들어 놓은 phpmyadmin의 설정파일을 복사한다.
    cp /tmp/config.inc.php /var/www/html/phpmyadmin/
    # php문법을 해석해서 html로 만들기 위한 php-fpm을 실행한다.
    /usr/sbin/php-fpm7
    # 웹서버를 열어준다.
    nginx -g 'pid /tmp/nginx.pid; daemon off;'
    ```

## Step 3 : config.inc.php 파일 작성

- phpmyadmin의 기본 설정파일이다.
    - 주된 목적으로, mysql과 연동을 위해 사용한다.

    `vim config.inc.php`

    ```jsx
    <?php
    declare(strict_types=1);

    $cfg['blowfish_secret'] = 'i0ULtq/xlL:x;F7AjIYH=2I82cI[Vrly'; /* YOU MUST FILL IN THIS FOR COOKIE AUTH! */
    $i = 0;
    $i++;
    $cfg['Servers'][$i]['auth_type'] = 'cookie';
    # 이 부분에 mysql의 host를 넣어준다.
    $cfg['Servers'][$i]['host'] = 'mysql-service';
    $cfg['Servers'][$i]['compress'] = false;
    $cfg['Servers'][$i]['AllowNoPassword'] = false;
    $cfg['PmaAbsoluteUri'] = '/phpmyadmin/';

    $cfg['UploadDir'] = '';
    $cfg['SaveDir'] = '';
    ```

    - 여기서, host 부분에 mysql의 서비스 이름만 넣어줘도 된다.
    - 그 이유는 이 [페이지](https://www.notion.so/Kubernetes-DNS-a3b707350c324444878d918543a0e65d)에 있다.

## Step 4: default.conf 작성

- nginx의 설정파일인 default.conf를 작성해준다.

    `vim default.conf`

    ```jsx
    # This is a default site configuration which will simply return 404, preventing
    # chance access to any other virtualhost.

    server {
    	listen 5000 default_server;
    	listen [::]:5000 default_server;

    	root /var/www/html/phpmyadmin;

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

    - 이 전까지 과제를 잘 따라왔다면 해석하는 데 문제는 없을거라 생각한다.

## Step 5 : phpmyadmin.yaml파일 작성

- 실제로 k8s에 연결하기 위한 phpmyadmin.yaml을 작성해준다.

    `vim phpmyadmin.yaml`

    ```jsx
    apiVersion: v1
    kind: Service
    metadata:
        name: phpmyadmin-service
        labels:
          type: app
          service: phpmyadmin
    spec:
        ports:
        - port: 5000
          protocol: TCP
          targetPort: 5000
        selector:
            type: app
            service: phpmyadmin
        type: LoadBalancer

    ---

    apiVersion: apps/v1
    kind: Deployment
    metadata:
        name: phpmyadmin-deployment
        labels:
            type: app
            service: phpmyadmin
    spec:
        selector:
            matchLabels:
                type: app
                service: phpmyadmin
        strategy:
            type: Recreate
        template:
            metadata:
                labels:
                    type: app
                    service: phpmyadmin
            spec:
                containers:
                - image: phpmyadmin-image
                  imagePullPolicy: Never
                  name: phpmyadmin-container
                  ports:
                  - containerPort: 5000
                  livenessProbe:
                    httpGet:
                      port: 5000
                    initialDelaySeconds: 5
                    periodSeconds: 5
    ```

- 이제, 모든 작업이 끝났다면 다음과 같은 과정을 거치면 연결이 된다.
    - 단, 사전에 [mysql pod](https://www.notion.so/MySQL-in-alpine-d02b8abcb9e14c9ba8fe7ad8c8cdecea)이 실행되고 있어야 정상작동된다.
    - `eval $(minikube docker-env)`
    - `docker build -t phpmyadmin-image .`
    - `kubectl apply -f phpmyadmin.yaml`
