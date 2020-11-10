# MySQL in alpine

- 먼저, [metalLB](https://www.notion.so/MetalLB-6d06bb696fa540cbbe67678fb960063c) 가 설치되어 있는지 확인하자.
<details>
    <summary>우선, Docker 환경에서 실행이 잘 되는지 확인해보자. (필수사항 아님)</summary>
- 먼저, Dockerfile을 작성해준다.

`vim Dockerfile`

```jsx
FROM alpine:latest

RUN apk update
RUN apk add mysql

# p 옵션은 중간에 폴더가 없을 때 자동으로 생성
# mysqld 폴더를 만들어 준다. ?역할은 잘 모름?
RUN mkdir -p /run/mysqld
# mysqld를 실행할 환경을 구성해준다.
COPY my.cnf /etc/mysql/
# mysql를 최초로 실행했을 때 유저와 비밀번호를 정해준다.
COPY mysql-init /tmp/
# 모든 명령어가 run.sh에서 시작된다.
COPY run.sh /tmp/

# expose는 굳이 안해줘도 되지만(k8s에서 자동으로 해준다) Dockerfile에 명시해주기 위해 적음.
EXPOSE 3306

# ENTRYPOINT와 CMD의 차이?
ENTRYPOINT["sh", "/tmp/run.sh"]
```

- my.cnf 파일을 작성해준다.

    `vim my.cnf`

    ```jsx
    [mysqld]
    # user는 root
    user=root
    port=3306
    #데이터가 저장될 위치 (Persistent Volume)
    datadir=/var/lib/mysql
    # mysql을 통해 실행한 명령어들의 모음.
    # 이걸 이용해 해킹할 수도 있겠다.
    log-bin=/var/lib/mysql/mysql-bin
    # 외부와의 연결 통로. 0.0.0.0은 모든 ip의 접근을 허용하겠다는 뜻.
    bind-address=0.0.0.0
    #만약 1이면 로컬에서만 수신한다.
    # 다른 컨테이너에서 접근하게 하고싶으면 0이라 적는다
    skip-networking=0
    ```

- mysql-init 파일을 작성해 준다.

    `vim mysql-init`

    ```jsx
    flush privileges;
    #모든 권한을 admin 그리고 % 의 의미는 모든 아이피에서 접근을 허용한다는 뜻이다.
    #identified by 는 비밀번호고 그 다음에 올 것은  권한 부여이다. 
    #세미콜론을 붙이지 않으면 제대로 작동하지 않는다....ㅅㅂ
    grant all privileges on *.* to 'admin'@'%' identified by '1q2w3e4r' with grant option;
    flush privileges;
    ```

- [run.sh](http://run.sh) 파일을 작성해준다.

    `vim run.sh`

    ```jsx
    #!/bin/sh

    # apk add mysql을 하면 mysql이 설치는 되지만 "mysql server"에 관련된 초기 세팅은 전혀 되어있지 않은 상태다. 따라서 서버에 관한 초기세팅을 해줘야하는데 mysql_install_db가 이 세팅을 도와준다. 그리고 --user=root로 하는 이유는 alpine 컨테이너에 우리가 다른 사용자를 만들지 않았기 때문. 그리고 만약 다른 사용자를 추가해서 그 유저로 하면 비밀번호를 추가로 입력해야 하는 불편함이 있다. 따라서 그냥 root로 편하게 하자.
    mysql_install_db --user=root
    # bootstrap옵션을 사용해 주는 이유는 mysql 서버가 시작되기 전에 먼저 DB 테이블이 만들어져야 하기 때문. 테이블이 만들어져 있지 않은 상황에서 wordpress.sql 데이터가 들어 갈 수 없다. 따라서 wordpress.sql을 해주기 전에 sql 테이터가 입력될 환경을 조성해 주는 것.
    # 그리고 mysqld --user=root < init 이 아니라 --bootstrap 옵션으로 먼저 테이블을 생성하고 mysqld --user=root를 하는 이유는 mysqld --user=root는 "서버"를 시작하는 명령이기 때문이다. 서버를 시작하고 나서는 데이터를 넣기 힘들기 때문에 서버를 시작하기 전에 "테이블"을 생성하고 만든 테이블을 갖는 서버를 시작하는 것.
    # --bootstrap 옵션을 붙여주는 이유는 결국 서버를 "진짜"로 시작하기 전에 테이블을 만들기 위함
    mysqld --user=root --bootstrap < /tmp/mysql-init
    # 서버 시작. 서버가 돌아가는 와중에 이제 만들어진 wordpress 테이블에 wordpress.sql 데이터가 들어온다. 이 작업은 wordpress.sql에서 해준다.
    mysqld --user=root
    ```

- Docker Container를 위한 Makefile 생성

    `vim Makefile`

    ```jsx
    IMG_NAME	=	my_mysql
    PS_NAME		=	mysql_ps
    PORT		=	3306

    all	:	build run

    run	:
        docker run --name $(PS_NAME) -d -p $(PORT):$(PORT) $(IMG_NAME)

    runit:
        docker run --name $(PS_NAME) -it -p $(PORT):$(PORT) $(IMG_NAME)

    exec :
        docker exec -it $$(docker ps -aq -f "name=$(PS_NAME)") sh

    build	:
        docker build -t $(IMG_NAME) .

    rm	:
        docker rm -f $$(docker ps -f "name=$(PS_NAME)" -aq)

    rmi	:
        docker rmi -f $(IMG_NAME)
    ```

- 이제, 실행해보자.

    `make`

    `docker ps 명령어로 컨테이너 생성 확인, 컨테이너 ID 확인`

    - 여기서 container의 ID를 확인해야하는 이유는 42 seoul cluster에선 이 컨테이너에서의 접속을 확인할 방법을 찾지 못했기 때문이다.
        - 그 이유는 brew install mysql-client 명령어가 먹지 않기 때문이다..
        - 또 클러스터에서 설치하게되면 용량을 잡아먹을 수 있다.
    - 따라서, docker로 다른 컨테이너를 띄우고 그 컨테이너에서 mysql 접속을 시도해볼 수 있었다.
    - 그러려면, mysql의 container IP (ID 아님) 를 알아야 한다.
    - container IP Address를 얻기위해 다음과 같은 명령어를 입력해보자.

        `docker inspect <container ID> | grep IPAddress`

        - 참고로, container ID는 앞의 네 자리만 입력해도 된다. git checkout과 마찬가지로.
    - 연결 되었다면, [nginx 컨테이너](https://www.notion.so/Nginx-and-SSH-SSL-protocol-in-alpine-13cf9068e2d24408bee140212919e855) 하나를 다른 터미널에 띄우자. (맨 위에서 세번째줄 토글 안에 있다.)
    - 그리고 나서, nginx 컨테이너 안으로 들어가서 다음 명령어를 입력해주자

        `apk add mysql-client`

        ![MySQL%20in%20alpine%20ebb7d5e758614141b7391d270cf8a718/Untitled.png](MySQL%20in%20alpine%20ebb7d5e758614141b7391d270cf8a718/Untitled.png)

        `/ # mysql -h <container_IP> -P 3306 -u admin --password=1q2w3e4r`

        ![MySQL%20in%20alpine%20ebb7d5e758614141b7391d270cf8a718/Untitled%201.png](MySQL%20in%20alpine%20ebb7d5e758614141b7391d270cf8a718/Untitled%201.png)

        - 위와 같이 뜨면 접근 성공.
            
</details>

---

## Step 1 : Dockerfile 작성하기

`vim Dockerfile`

```jsx
FROM alpine:latest

RUN apk update
RUN apk add mysql

RUN mkdir -p /run/mysqld
COPY my.cnf /etc/mysql/
COPY mysql-init /tmp/
COPY run.sh /tmp/

EXPOSE 3306

ENTRYPOINT ["sh", "/tmp/run.sh"]
```

---

## Step 2 : my.cnf 파일 작성하기

- my.cnf는 mysqld의 설정파일이다.

`vim my.cnf`

```jsx
[mysqld]
# user는 root
user=root
port=3306
# 데이터가 저장 될 위치. PV 위치.
datadir=/var/lib/mysql
log-bin=/var/lib/mysql/mysql-bin
# log file 저장 경로
log-error=/var/log/mysqld.log
# 외부와 연결하기 위한 통로. 0.0.0.0의 의미는 모든 ip를 허용하겠다는것.
# 그냥 하나의 컨테이너에서 mysql, phpmyadmin 등을 돌리는거면 127.17.0.1 뭐 이런식으로 하면됨
# 그냥 무조건 0.0.0.0 쓰는게 좋음
bind-address=0.0.0.0
# 만약 1 이면 로컬에서만 수신한다. 따라서 0으로 바꿔줌.
# 다른 컨테이너에서 접근해야하므로.
skip-networking=0
```

---

## Step 3 : mysql-init (sql 명령어 모음) 작성하기

- mysqld의 유저 및 권한을 주기 위해서 SQL명령어를 모은 파일을 따로 만들어 준다.
- 그리고 wordpress에서 mysql에 접근하여야 하기 때문에 데이터베이스도 만들어 준다.

`vim mysql-init`

```jsx
FLUSH PRIVILEGES;
CREATE USER 'modyhoon'@'%' IDENTIFIED BY '1q2w3e4r';
GRANT ALL PRIVILEGES ON *.* TO 'modyhoon'@'%' IDENTIFIED BY '1q2w3e4r' WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%' IDENTIFIED BY '1q2w3e4r' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

- 주의할 점은 세미콜론을 빼먹으면 명령어가 제대로 실행되지 않는다.
- 또한, SQL 문이 하나라도 정상적이지 않을 경우, 모든 SQL문의 실행은 중단된다. 따라서 오류가 난다.

---

## Step 4 : run.sh 작성하기

- mysqld를 실행할 shell script를 작성해 준다.

`vim run.sh`

```jsx
mysql_install_db --user=root
mysqld --user=root --bootstrap < /tmp/mysql-init
mysqld --user=root
```

---

## Step 5 : Deployment 및 Service 배포하기

- Service와 Deployment를 명세하는 .yaml 파일을 만들어야 한다.

`vim deployment.yaml`

```jsx
apiVersion: v1
kind: Service
metadata:
    name: mysql-service
    labels:
      type: app
      service: mysql
spec:
    ports:
    - port: 3306
      protocol: TCP
      targetPort: 3306
    selector:
        type: app
        service: mysql
    type: ClusterIP

---

apiVersion: apps/v1
kind: Deployment
metadata:
    name: mysql-deployment
    labels:
        type: app
        service: mysql
spec:
    selector:
        matchLabels:
            type: app
            service: mysql
    strategy:
        type: Recreate
    template:
        metadata:
            labels:
                type: app
                service: mysql
        spec:
            containers:
            - image: mysql-image
              imagePullPolicy: Never
              name: mysql-container
              env:
              - name: MYSQL_ROOT_PASSWORD
                value: password
              volumeMounts:
              - name: mysql-persistent-storage
                mountPath: /var/lib/mysql
            volumes:
            - name: mysql-persistent-storage
              persistentVolumeClaim:
                  claimName: mysql-pv-claim
```

---

## Step 6 : Persistent Volume 그리고 Persistent Volume Claim 작성하기

- Persistent Volume(이하 PV) 는 볼륨에 대한 명세를, 그리고 Persistent Volume Claim은 그 볼륨을 직접 할당 (요청)하는 역할을 한다.
- 자세한 설명은 [여기](./volume_pv_pvc/README.md) 에 있다.

```jsx
apiVersion: v1
kind: PersistentVolume
metadata:
    name: mysql-pv-volume
    labels:
        type: local
spec:
    storageClassName: manual
    capacity:
        storage: 1Gi
    accessModes:
        - ReadWriteOnce
    hostPath:
        path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: mysql-pv-claim
spec:
    storageClassName: manual
    accessModes:
        - ReadWriteOnce
    resources:
        requests:
            storage: 1Gi
```

---

## Step 7 : K8S에 적용하기

```jsx
eval $(minikube docker-env)
docker build -t "mysql-image" .
kubectl apply -f deployment.yaml
kubectl apply -f pv.yaml
```
