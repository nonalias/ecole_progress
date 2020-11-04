# vsftpd in alpine

## 이 페이지는 alpine환경에서 vsftpd를 사용하기 위한 과정을 기술하였음.

<details>
    <summary>먼저, k8s 환경이 아닌, Docker Container로 vsftpd 환경을 만들어 보자. **(필수사항 아님)**</summary>
    
- 도커로 alpine 컨테이너에 들어가자.
    - `docker run -it -p 21:21 -p 50000:50000 alpine:latest`
        - 21번 포트와 50000번 포트를 개방하면서 dockerhub에 있는 alpine 이미지를 불러와 -it 옵션으로 그 컨테이너 안에 들어가는 명령.
        - 50000번 포트를 개방해놓는 이유는, passive 연결을 위해서이다.
        - 성공적으로 접속한 모습

            ![vsftpd%20in%20alpine%20b1e7822171f24b098d1b943a2860807b/Untitled.png](vsftpd%20in%20alpine%20b1e7822171f24b098d1b943a2860807b/Untitled.png)

- vsftpd를 설치하고 vsftpd.conf 파일을 수정하자.

    ```cpp
    **(여기서 / # 은 shell이라는 것을 알려주기 위한 것이다. 실제로 입력할 땐 지우자!)**
    / # apk update
    / # apk add vsftpd

    vim /etc/vsftpd/vsftpd.conf

    모두 지우고, 아래 내용을 복붙한다.

    anonymous_enable=NO
    local_enable=YES
    write_enable=YES
    local_umask=022
    dirmessage_enable=YES
    xferlog_enable=YES
    connect_from_port_20=YES
    allow_writeable_chroot=YES
    chroot_local_user=YES
    passwd_chroot_enable=YES
    listen=YES
    pam_service_name=vsftpd
    seccomp_sandbox=NO

    pasv_enable=YES
    pasv_min_port=50000
    pasv_max_port=50000
    ```

- 저장한 후 다음 명령어를 실행시켜 준다.

    ```cpp
    / # USER=admin
    / # PASSWORD=admin
    / # mkdir -p /ftps/$USER
    / # addgroup -g 433 -S $USER
    / # adduser -u 431 -D -G $USER -h /ftps/$USER -s /bin/false $USER
    / # echo "$USER:$PASSWORD" | chpasswd
    / # chown -R $USER:$USER /ftps/$USER
    ```

- 이제, vsftpd를 실행시켜보자 !
    - `/ # vsftpd /etc/vsftpd/vsftpd.conf`
- 입력한 후 멈춰 보이는 것은 이상한 것이 아니다. 잘 실행 되고 있는 것이다. 그 상태로 **다른 터미널 하나를 띄워** 다음 명령어를 입력해 보자.
- 파일을 전송해보자
    - `> curl ftp://localhost:21 -u admin:admin -T <your_any_file>`
        - T 옵션은 transfer로 추정되며, 파일을 해당 주소로 보낸다.
        - 여기서 your_any_file은 아무 파일이나 경로를 적절히 넣어주면 된다.
- 잘 도착했는지 확인해보자.
    - `> curl ftp://localhost:21 -u admin:admin`

        ![vsftpd%20in%20alpine%20b1e7822171f24b098d1b943a2860807b/Untitled%201.png](vsftpd%20in%20alpine%20b1e7822171f24b098d1b943a2860807b/Untitled%201.png)

        - 다음과 같이 리스트가 뜨면 성공.
- 파일을 받아 오려면?
    - `> curl ftp://localhost:21/<uploaded_file> -u admin:admin -o ./<set_proper_name>`
    - 여기서 uploaded file은 해당 폴더에 있는 파일이다.
    - set_proper_name 은 그 파일을 어떤 이름으로 저장할지 정해준다. gcc 에서 -o옵션이랑 비슷하다.
        
</details>

---

### Step 1: Dockerfle 작성

```cpp
cd srcs/ftps
vim Dockerfile

FROM alpine:latest # alpine 의 최신버전을 받아옴

RUN apk update # update를 하지 않으면 apk가 vsftpd를 찾지 못함.
RUN apk add vsftpd # vsftpd 설치,
# 이제 /usr/sbin/vsftpd가 생겼을 것이다.

ENV USER=modyhoon # ftp container 에서 사용할 User id (바꿀수 있다)
ENV PASSWORD=1q2w3e4r # 위와 동일

# vsftpd의 conf 파일을 /etc/vsftpd/ 경로로 복사.
# 미리 작동되게끔 만들어 놓고, 로컬에 있는 vsftpd.conf를
# container 안의 /etc/vsftpd/ 경로로 복붙한다.
# 추가로, alpine에서 vsftpd의 default conf 경로는
# /etc/vsftpd/vsftpd.conf 이다.
COPY ./vsftpd.conf /etc/vsftpd/

# 코드를 간결화 하기 위해 실행부분을 .sh파일로 나누어 놨다.
# 추가적으로, livenessprobe를 보장하기 위해서이기도 하다.
COPY ./ftp.sh /tmp/ftp.sh # 복사될 경로는 어디라도 상관없다.
# ssl 연결을 위해 미리 만들어놓은 key 값을 복사.
COPY ./vsftpd.pem /etc/ssl/private/vsftpd.pem

# 외부와 통하는 포트를 열어준다.
EXPOSE 21

# shell에서 'sh /tmp/ftp.sh' 와 같다.
# 참고로, livenessProbe는 이 CMD로 실행된 명령 이전에 실행된
# 응용 프로그램들의 생존은 확인하지 않는다. 
CMD ["sh", "/tmp/ftp.sh"]
```

- 이 Dockerfile은 ftp의 image를 만드는 데 사용된다.

---

### Step 2 : ftp.sh 작성

- 앞서 말했듯이, shell 파일은 Dockerfile의 코드 수를 줄여주고, Dockerfile에서 CMD로 실행된 명령 이전의 프로그램들은 livenessProbe를 체크하지 않는다.
- 따라서, k8s가 vsftpd가 살아있는지 확인하는 절차는 이 CMD 명령으로 실행된 명령어에서 확인한다.

```cpp
# ftp 연결을 할 폴더를 생성해준다.
# 여기서 p 옵션은 만약 중간 경로에 폴더가 존재하지 않을경우, 만들어준다.
mkdir -p /ftps/$USER
# group을 추가해 준다. 여기서 $USER는 Dockerfile에서 ENV 명령어로 만든
# 그 USER이다.
addgroup -g 433 -S $USER
# user를 생성해 준다.
adduser -u 431 -D -G $USER -h /ftps/$USER -s /bin/false $USER
# 말 뜻 그대로 $USER:$PASSWORD의 내용을 바탕으로 유저의 비밀번호를 바꿔준다.
echo "$USER:$PASSWORD" | chpasswd
# ftp 연결을 할 폴더의 소유권을 $USER로 바꾸어준다.
# 여기서 R 옵션은 Recursive로, 하위폴더 내 하위폴더 및 파일들을 전부 바꿔준다.
chown -R $USER:$USER /ftps/$USER

# sbin은 명령어들이 저장되어있는 곳이다.
# 그냥 vsftpd를 입력해도 무관하지만, 직관성을 얻기 위해 적었다.
# 뒤에있는 인자는 해당 conf 파일로 vsftpd를 실행하겠다는 뜻이다. 
# 아마 conf 를 지정해주는 걸로 보아 conf의 위치는 크게 상관이 없는 것 같다.
/usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
```

---

### Step 3 : vsftpd.conf 작성

- 앞서 말했듯이, apk add vsftpd를 하면 /etc/vsftpd/vsftpd.conf 가 기본적으로 생성된다.
<details>
    <summary>맨 처음 기본으로 들어있는 내용은 다음과 같다.  (보고싶으면 보자)</summary>

    ```cpp
    anonymous_enable=YES
    dirmessage_enable=YES
    xferlog_enable=YES
    connect_from_port_20=YES
    listen=YES

    # 참고로, 주석을 모두 제거한 내용이다.
    ```
</details>

- 이 코드에 필요한 내용들을 추가하면 다음과 같다. (참고 : [https://faq.hostway.co.kr/Linux_ETC/1474](https://faq.hostway.co.kr/Linux_ETC/1474))

```cpp
local_enable=YES # 로컬 계정 사용자 접속 여부 설정 (**필수**)
seccomp_sandbox=NO (필수)

anonymous_enable=NO # 익명의 user가 접근할 수 있는지 물어보는 코드 (없어도 되지만, 보안상 넣는걸 추천, 기본값 NO인듯)
write_enable=YES # 쓰기 가능 여부 설정 (없어도 되지만, 넣는 것 추천)
local_umask=022 # 파일 퍼미션 (접근권한) 정의 (022로 설정하면 파일의 퍼미션은 644가 됨) (넣는거 추천)
allow_writeable_chroot=YES # (없어도 됨)
chroot_local_user=YES # 홈 디렉토리 위로 이동 제한 여부 설정 (기본값 NO) (없어도 됨)
passwd_chroot_enable=YES # (없어도 됨)
pam_service_name=vsftpd # pam 인증에 사용할 설정파일의 이름 설정 (없어도 됨)
dirmessage_enable=YES # FTP 접속자가 다른 디렉토리로 이동시, 알림메시지 출력 여부 (없어도 됨)
xferlog_enable=YES # FTP 접속자들의 업/다운로드 상황 로그파일 저장 여부 (없어도 됨)
connect_from_port_20=YES # standalone 모드를 운영하면서 데이터 전송포트 사용시 설정 (없어도 됨)
listen=YES # standalone 모드로 서비스할 때 설정 (없어도 됨)

pasv_enable=YES # 패시브 모드 설정
pasv_min_port=50000
pasv_max_port=50000

# 이하 ssl 연결을 위한 설정
rsa_cert_file=/etc/ssl/private/vsftpd.pem
rsa_private_key_file=/etc/ssl/private/vsftpd.pem
ssl_enable=YES
allow_anon_ssl=NO
force_local_data_ssl=YES
force_local_logins_ssl=YES
ssl_tlsv1=YES
ssl_sslv2=NO
ssl_sslv3=NO
require_ssl_reuse=NO
ssl_ciphers=HIGH
```

---

### Step 4 : ftp.yaml 파일 생성

- 이제, docker image를 통해 k8s와 연결할 준비만 남았다!
- 그러기 위해서는 Deployment를 배포하고 그 Deployment에 대응하는 서비스를 expose 해주면 된다.
- 직접 명령어를 입력한다면 수많은 명령어가 필요할 것이고, 인자 또한 많아질 것이다.
- 한번 해보고 만다면 명령어를 입력하는 것이 좋을 수(?) 도 있겠지만, 당연히 첫 try 만에 성공할 리도 없고, 이 코드는 계속 써먹을것이 분명하다.
- 따라서, 그러한 반복작업을 줄여주는것이 yaml파일이다.

    ```cpp
    vim ftp.yaml

    아래 내용을 복붙하자.

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: ftps-deployment
      labels:
        type: app
        service: ftps
    spec:
      #replicas: 2
      selector:
        matchLabels:
          type: app
          service: ftps
      template:
        metadata:
          labels:
            type: app
            service: ftps
        spec:
          containers:
          - name: ftps-container
            image: ftps-image
            imagePullPolicy: Never
            livenessProbe:
              initialDelaySeconds: 20
              periodSeconds: 10
              timeoutSeconds: 5
              tcpSocket:
                port: 21
            ports:
              - containerPort: 21
                hostPort: 21
              - containerPort: 50000
                hostPort: 50000

    ---

    apiVersion: v1
    kind: Service
    metadata:
      name: ftps-service
      labels:
        type: app
        service: ftps
    spec:
      type: LoadBalancer
      ports:
        - name: ftps
          port: 21
          protocol: TCP
          targetPort: 21
        - name: passive
          port: 50000
          protocol: TCP
          targetPort: 50000
      selector:
        type: app
        service: ftps
    ```

- 자세한 설명은 적지 않겠다. 참고할만한 사이트는 다음과 같다.
- [https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/kubernetes-objects/](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/kubernetes-objects/)

---

### Step 5 : SSL 작업을 위한 key 생성

- 직접 openssl을 통해 key를 만들 수 있겠지만, 귀찮아서 미리 만들어놓은 key를 사용한다.

    ```cpp
    vim vsftpd.pem

    아래 내용을 복붙한다.

    -----BEGIN PRIVATE KEY-----
    MIIEvAIBADANBgkqhkiG9w0BAQEFAASCBKYwggSiAgEAAoIBAQDB/IS8sinqexrK
    QEdooHyRbwIJ+pRiTzDw9Fe6jipU0MHbkgKiywZo76JBuuwBgJ10Oz0/Qy/Refj5
    p0h7aCt3Ku17EoN3aQGBaDel7TFs0YK1h+ECLcVWQlUic1uJZElhfpP2OQlcOGdS
    t/Md5f5c8K2JWZYF9BaQcmiCDn+ALqP0PioxD0NQfWDPVXGMCUZxF30Q0zoyc72T
    +FjfMrHS7mtUi/Y++Plrt5ITsIt8LG1Apg+C16g8C439MihRA0uzLs6PQYUp1jZV
    Er4BU2ijt+SuwPQOwdEe3lWeLm93lnWvC5XjANGxjuecm+a9Vc4ha55E6aU9eCzz
    MRcwfPglAgMBAAECggEAVOmC9NIL9P6j8GoIl+y/+i0cOF/+ObYuVxqtmBSIxQ2H
    /ePA0Z+LE73pSVpX2iSBR5JysdFoCgqZCDbITHSBqi0ZPKkS8N7+8LU8vp2/58Eh
    tPJgdMKyQZrRhz31kINcd5efjsTSqxJpb9TjT3AQUoBrhda4C60Xf20FAAD2oJWG
    wntSs/dqw0Zb830+O98VD7gnZOR5vZJY5Bx8R2FeY6NZM4ufQmZzJdlkcv22Nvq1
    uioboZ6l+fa5PTZ5H0vDyUi6zaA4ozGvpbUo/hEUGXdjNlAlVKWdIVYM9mtT71/0
    N09g1RvCpai9mYWuZQ8Dc3lx1iMBgKzNTGZ3HUz9NQKBgQDiihR3lNGNa7TAkkSN
    XJ6gy4pwp480USbzINI6+GVhb8H5tk30pbxqTHt5GXk6hqvmVIWzPZvRMla046+5
    nwXyrTrm5nf+5bgLcsysmno9W7sDkyBwpE2uaM2kupgLFUF5gzqxODXBPuSxf+a6
    Zh/ucRKLksioT1/0Bjptk0eVDwKBgQDbNrBnASI1n89MiuhO0yiQ76KiZ1tvAxyi
    IKTgmr8pqqEwEGhDPhu21vJCLz/YfTMY0gQ77sI19wXAFiXTeZdx6V1pVFJ+v5qa
    lLMu5vNHZ4wcX81r7hKikHwq9HN4QFnKLVYwNCK+aB9O+w/k86bYiVVkw4XgPhg9
    jirBdfpniwKBgGNa2eUkYM+ciFbZD7XMBEpTWrFT28u/N8zz/SAd5yDXygRB/2in
    873PM2wGTxPrEqNfOJBHGfqjEEIfhedsJkirzySLud8SUyi6PagJzEjy3U+RDG46
    sVMn5eE0cRCTTvcDJg+prnHFqrlqdgAUYDbMYqzSQK0IuvWkcaWzLXbjAoGAcGvp
    p8mzC6E7pNuQK+yq3zmmRHeRMqt74cGwDOgPpYS2SXoAnouZlvlBIKQusA31SINc
    XIgj3Z0ju9Ef8QZonqi5mSz/abVFyoT8J8+VcEcwWdTf+rwLnodOxpC7Ly6BXehG
    TU5PiyrG87BaBGbYaDB2NMj5PXla4Sap0rF4i+UCgYAXuXaxnGXKkf1tH0CQZPMO
    j/sNpLRgEgzkg5HVBOCqDdLFfjpqQa2noN2WyUZ2Vw/sYaIsRaPJDgOL0s5VV0tF
    t0CqPs0jP/JlslFA+lTtKD5LNncp5E7hHWFwD8BDcjEwVs/FdI2wqqceXPa6eIfZ
    MCp4bzxUzYM/fTv8D3FmPw==
    -----END PRIVATE KEY-----
    -----BEGIN CERTIFICATE-----
    MIICyDCCAbACCQC6m0qo+BvDLDANBgkqhkiG9w0BAQsFADAmMREwDwYDVQQDDAhu
    Z2lueHN2YzERMA8GA1UECgwIbmdpbnhzdmMwHhcNMjAwNzI3MDEwNTAyWhcNMjEw
    NzI3MDEwNTAyWjAmMREwDwYDVQQDDAhuZ2lueHN2YzERMA8GA1UECgwIbmdpbnhz
    dmMwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDB/IS8sinqexrKQEdo
    oHyRbwIJ+pRiTzDw9Fe6jipU0MHbkgKiywZo76JBuuwBgJ10Oz0/Qy/Refj5p0h7
    aCt3Ku17EoN3aQGBaDel7TFs0YK1h+ECLcVWQlUic1uJZElhfpP2OQlcOGdSt/Md
    5f5c8K2JWZYF9BaQcmiCDn+ALqP0PioxD0NQfWDPVXGMCUZxF30Q0zoyc72T+Fjf
    MrHS7mtUi/Y++Plrt5ITsIt8LG1Apg+C16g8C439MihRA0uzLs6PQYUp1jZVEr4B
    U2ijt+SuwPQOwdEe3lWeLm93lnWvC5XjANGxjuecm+a9Vc4ha55E6aU9eCzzMRcw
    fPglAgMBAAEwDQYJKoZIhvcNAQELBQADggEBAKzf2oJfpy0IkvGw8Kn0TdsKucOy
    3c4PmHpkgCrZG+q68wP2VQpdQLiyXTQK7XbseYz2aHE6/zmfLlodzvI8BAZ89Cuy
    P8eDaod8eVO4900ofNFVCwpjHQfED1xIUy59QnzXYiAKQ60esYUTuN+MSnKAMWfP
    lzgqMlh4uAcFKcSyfqxALiZE3kqHzhuJ93F47a+5m5sYceWv6OXbljBmfD+8jdCD
    UPSpICNT1lOdutk4pJsN2Ag3GtZ34r0ttZ90RPmwz0gVazx1Ews+vXMWxtPWsK3u
    lg8jpUCxv1k13COhIgaQIP6ZnWzaa+rW6tOkoH8YaJeFZR/Nsv3t4BV9N1E=
    -----END CERTIFICATE-----
    ```

---

### Step 6 : k8s에 적용하기

- 앞서, nginx나 기타 다른 과제를 먼저 진행했다면 알아서 진행하리라 믿는다.
- 그렇지 않고 k8s를 처음 써본다면 다음 을 참고하자.
    - `minikube start --driver=virtualbox`
    - `eval $(minikube docker-env)`
    - `docker build -t ftps-image .`
    - `kubectl apply -f ftp.yaml`
