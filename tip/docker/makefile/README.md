# Docker Image Makefile로 자동화하기

- Services를 하던 중, Docker로 먼저 띄워서 성공하는지 확인해 보고 그 다음 yaml 작업을 하는 편이다.
- 그런데, 매번 변경사항이 있을 때마다 docker 명령어를 쳐줘야되는 번거로움이 있다.
- 그러한 점을 해결하기 위해 Makefile을 하나 만들어서 간편하게 사용하려 한다.

---

## 폴더 분리

- 먼저 Makefile은 Service를 k8s에 연결할 때 사용되지 않는다. 가시성도 좋지 않고 다른 사용자에게 하여금 혼동을 줄 수 있으므로 local_not_necessary라는 폴더를 하나 만들어서 그 안에 넣도록 하겠다.

`mkdir local_not_necessary`

`cd local_not_necessary`

- 그 다음, Docker Image에 필요한 자료들 (dockerfile, .sh파일이라든지, conf 파일이라든지.)을 Makefile이 읽을 수 있게 해야한다.
- Makefile설정을 잘 건드리면 home 경로를 바꿀 수 있겠지만, 나는 **하드링크**를 하는 방법을 택했다.
- 여기서 하드링크란, 심볼릭링크와는 다르게 특정 파일의 내용을 그대로 보존하여 파일을 만든다.
    - cp 와 같은 개념이라고 생각할 수 있는데, cp는 파일을 복제하면서 서로 다른 inode를 갖지만,
    - 하드 링크는 서로 같은 inode를 가진다. 따라서 어느 한 파일을 수정해도 같은 inode를 공유하고 있다면 덩달아 그 파일의 내용도 수정된다.
    - 만약 같은 inode를 가진 두 파일이 있다. 여기서 한 파일을 제거하더라도 그 파일에 영향은 주지 않는다.
    - 요약하자면, **내용은 서로 공유하지만 파일 자체는 독립적**으로 작동한다는 뜻이다.
    - 사용법

        `ln <original file> <target file or link>`

```jsx
예제입니다.

ln Dockerfile ./local_not_necessary
ln default.conf ./local_not_necessary
ln nginx.sh ./local_not_necessary
```

---

## Makefile 작성

```jsx
# 재사용하기 위해 변수로 빼놓음
IMG_NAME	=	my_nginx
PS_NAME		=	nginx-ps
PORT1		=	80
PORT2		=	443
PORT3		=	22

# make를 입력하면 자동으로 image build와 container를 가동시켜줌
all	:	build run

# container를 실행시킴
run	:
	docker run --name $(PS_NAME) -d -p $(PORT1):$(PORT1) -p $(PORT2):$(PORT2) $(IMG_NAME)

exec:
	docker exec -it $$(docker ps -aq -f "name=$(PS_NAME)") sh

# Dockerfile을 기반으로 도커 이미지 생성
build	:
	docker build -t $(IMG_NAME) .

# container 종료 및 삭제
rm	:
	docker rm -f $$(docker ps -f "name=$(PS_NAME)" -aq)

# image 제거
rmi	:
	docker rmi -f $(IMG_NAME)
```
