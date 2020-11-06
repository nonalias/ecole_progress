# Nginx and SSH, SSL protocol in alpine

## Kubernetes(k8s)를 통해 nginx를 가동시켜 보자.

- 먼저, [MetalLB](Nginx%20and%20SSH,%20SSL%20protocol%20in%20alpine%2013cf9068e2d24408bee140212919e855/MetalLB%206d06bb696fa540cbbe67678fb960063c.md)가 설치되어있어야 한다.
- 시작하기에 앞서, Docker환경에서 nginx를 가동시켜 보자 (필수 아님)
    - 우선, Dockerfile을 작성해 준다.

        ```cpp
        vim Dockerfile

        FROM alpine:latest

        RUN apk update
        RUN apk add vim
        RUN apk add nginx
        RUN apk add openssl
        RUN apk add openssh

        # CMD 명령어로 실행할 sh 파일
        COPY ./nginx.sh /tmp/nginx.sh
        # nginx configuration, alpine nginx의 기본 경로는 /etc/nginx/conf.d/ 이다.
        COPY ./default.conf /etc/nginx/conf.d/default.conf

        # 80, 443, 22번 포트 개방
        EXPOSE 80 443 22

        # CMD명령어로 복사한 nginx.sh 실행
        CMD ["sh", "/tmp/nginx.sh"]
        ```

    - [nginx.sh](http://nginx.sh) 파일을 작성해준다.

        ```jsx
        vim nginx.sh

        echo 'root:1q2w3e4r' | chpasswd

        # nginx ssl connect setting
        openssl req -newkey rsa:4096 -days 365 -nodes -x509 -subj "/C=KR/ST=Seoul/L=Seoul/O=42Seoul/OU=KIM/CN=localhost" -keyout localhost.dev.key -out localhost.dev.crt
        mv localhost.dev.crt /
        mv localhost.dev.key /
        chmod 600 /localhost.dev.crt /localhost.dev.key

        # ssh setting
        sed 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' -i /etc/ssh/sshd_config
        ssh-keygen -f /etc/ssh/ssh_host_rsa_key -N '' -t rsa
        ssh-keygen -f /etc/ssh/ssh_host_dsa_key -N '' -t dsa

        /usr/sbin/sshd

        nginx -g 'pid /tmp/nginx.pid; daemon off;';
        ```

    - nginx 설정파일인 default.conf를 작성해준다.

        ```jsx
        vim default.conf

        # This is a default site configuration which will simply return 404, preventing
        # chance access to any other virtualhost.

        server {
        	listen 80 default_server;
        	listen [::]:80 default_server;

        	return 301 https://$host$request_uri;
        }
        server {
        	listen 443 ssl;
        	listen [::]:443 ssl;

        	ssl_certificate /localhost.dev.crt;
        	ssl_certificate_key /localhost.dev.key;
        	location / {
        		autoindex on;
        	}
        	root /var/lib/nginx/html;
        	index index.html;
        }
        ```

    - [Makefile을 작성](https://www.notion.so/Docker-Image-Makefile-6f13583a06a043e5b4938e040bb2a5b2)해준다.

        ```jsx
        IMG_NAME	=	my_nginx
        PS_NAME		=	nginx-ps
        PORT1		=	80
        PORT2		=	443
        PORT3		=	22

        all	:	build run

        run	:
        	docker run --name $(PS_NAME) -d -p $(PORT1):$(PORT1) -p $(PORT2):$(PORT2) -p $(PORT3):$(PORT3) $(IMG_NAME)

        exec:
        	docker exec -it $$(docker ps -aq -f "name=$(PS_NAME)") sh

        build	:
        	docker build -t $(IMG_NAME) .

        rm	:
        	docker rm -f $$(docker ps -f "name=$(PS_NAME)" -aq)

        rmi	:
        	docker rmi -f $(IMG_NAME)
        ```

    - `make run`
    - nginx 작동 확인
        - 80번포트
            - curl [localhost:80](http://localhost:80) -L -k
        - 443번포트
            - curl [https://localhost:443](https://localhost:443) -k
        - ssh
            - ssh root@localhost -u root:1q2w3e4r
            - 만약 ssh 실행시 certificate 문제가 발생한다면, 다음 명령어를 입력하고 다시 연결을 시도해보자.
                - `ssh-keygen -R localhost`

---

## STEP 1 : Dockerfile 작성

- local로 Docker Image를 만들어야 하기 때문에 Dockerfile을 작성해준다.

    ```jsx
    vim Dockerfile

    FROM alpine:latest

    RUN apk update
    RUN apk add vim
    RUN apk add nginx
    RUN apk add openssl
    RUN apk add openssh

    # CMD 명령어로 실행할 sh 파일
    COPY ./nginx.sh /tmp/nginx.sh
    # nginx configuration, alpine nginx의 기본 경로는 /etc/nginx/conf.d/ 이다.
    COPY ./default.conf /etc/nginx/conf.d/default.conf

    # 80, 443, 22번 포트 개방
    EXPOSE 80 443 22

    # CMD명령어로 복사한 nginx.sh 실행
    CMD ["sh", "/tmp/nginx.sh"]
    ```

## STEP 2: nginx.sh파일 작성

- [nginx.sh](http://nginx.sh) 파일을 작성해준다. 명령어를 독립적으로 운영할 수 있고, Dockerfile의 CMD 명령으로 인해 livenessProbe도 보장해줄 수 있다.

    `vim nginx.sh`

    ```jsx
    echo 'root:1q2w3e4r' | chpasswd

    # nginx ssl connect setting
    openssl req -newkey rsa:4096 -days 365 -nodes -x509 -subj "/C=KR/ST=Seoul/L=Seoul/O=42Seoul/OU=KIM/CN=localhost" -keyout localhost.dev.key -out localhost.dev.crt
    mv localhost.dev.crt /
    mv localhost.dev.key /
    chmod 600 /localhost.dev.crt /localhost.dev.key

    # ssh setting
    sed 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' -i /etc/ssh/sshd_config
    ssh-keygen -f /etc/ssh/ssh_host_rsa_key -N '' -t rsa
    ssh-keygen -f /etc/ssh/ssh_host_dsa_key -N '' -t dsa

    /usr/sbin/sshd

    nginx -g 'pid /tmp/nginx.pid; daemon off;';
    ```

## Step 3 : default.conf 작성

- nginx 설정파일인 default.conf를 작성해준다.

    `vim default.conf`

    ```jsx
    # This is a default site configuration which will simply return 404, preventing
    # chance access to any other virtualhost.

    server {
    	listen 80 default_server;
    	listen [::]:80 default_server;

    	return 301 https://$host$request_uri;
    }
    server {
    	listen 443 ssl;
    	listen [::]:443 ssl;

    	ssl_certificate /localhost.dev.crt;
    	ssl_certificate_key /localhost.dev.key;
    	location / {
    		autoindex on;
    	}
    	root /var/lib/nginx/html;
    	index index.html;
    }
    ```

## Step 4 : Deployment, Service를 위한 .yaml 파일 작성

- k8s에 적용하기 위해 yaml파일을 작성한다.
- 명령어의 옵션을 부여해 직접 실행할 수 있지만, 재사용성등을 고려해 yaml파일을 작성하였다.

    `vim deployment.yaml`

    ```jsx
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-deployment
      labels:
        type: app
        service: nginx
    spec:
      replicas: 2
      selector:
        matchLabels:
            type: app
            service: nginx
      template:
        metadata:
          labels:
              type: app
              service: nginx
        spec:
          containers:
          - name: nginx-container
            image: nginx-image
            imagePullPolicy: Never
            livenessProbe:
                initialDelaySeconds: 20
                periodSeconds: 10
                timeoutSeconds: 5
                tcpSocket:
                    port: 22
            ports:
                - containerPort: 80
                  name: http
                  protocol: TCP
                - containerPort: 443
                  name: https
                  protocol: TCP
                - containerPort: 22
                  name: ssh
                  protocol: TCP

    ---

    apiVersion: v1
    kind: Service
    metadata:
      name: nginx-service
      labels:
        type: app
        service: nginx
    spec:
      type: LoadBalancer
      ports:
        - name: http
          port: 80
          protocol: TCP
          targetPort: 80
        - name: https
          port: 443
          protocol: TCP
          targetPort: 443
        - name: ssh
          port: 22
          protocol: TCP
          targetPort: 22
      selector:
          type: app
          service: nginx
    ```

## Step 5 : k8s에 적용하기

- 앞서, nginx나 기타 다른 과제를 먼저 진행했다면 알아서 진행하리라 믿는다.
- 그렇지 않고 k8s를 처음 써본다면 다음 을 참고하자.
    - `minikube start --driver=virtualbox`
    - `eval $(minikube docker-env)`
    - `docker build -t nginx-image .`
    - `kubectl apply -f deployment.yaml`