# 42 Cluster에서 minikube 기본설정 하기

- ft_services가 처음이라면, 우선 minikube 먼저 셋팅을 해주어야 k8s를 쉽고 빠르게 사용할 수 있다.

---

## Step 1 : brew 설치

- minikube를 설치하기 위해서는 brew가 설치되어 있어야 한다.
- MacOS 의 패키지 관리자인 brew를 설치하기 위해서는 다음과 같은 명령어를 입력한다.
- `rm -rf $HOME/.brew && git clone --depth=1 https://github.com/Homebrew/brew $HOME/.brew && echo 'export PATH=$HOME/.brew/bin:$PATH' >> $HOME/.zshrc && source $HOME/.zshrc && brew update`

---

## Step 2 : virtualBox 설치

`command + space`

를 누르면 빠른 탐색 창이 나온다. 여기서 Managed Software Center를 입력하고 들어가자. (msc만 입력해도 나옴)

거기서 맨 왼쪽 탭으로 들어가서, 아래로 내리면 virtualBox가 보일 것이다. install 해주자.

만약 설치가 진행되지 않는다면, msc의 맨 오른쪽 탭으로 들어가서 update all을 해준다.

---

## Step 3 : Docker 설치

- ft_server를 진행했다면 Docker를 설치하는 방법은 알고 있을 것이다.
- MSC(Managed Software Center)에 들어가서 Docker 를 찾아 설치한다.
- 그 후, [42toolbox](https://github.com/alexandregv/42toolbox)를 clone 시켜서 bash init_docker.sh 를 해준다. 만약 비밀번호를 물어본다면 그냥 무시하고 엔터를 눌러준다. 중간에 Y/n 이 나오면 y를 입력해준다.
- 그리고 나서 우측 상단에 고래모양 배가 있는데, 고래가 물 뿌리는 것을 멈추면 실행이 완료된 것이다.

---

## Step 4 : Minikube 실행

- minikube를 실행하면 minikube 실행에 필요한 파일들을 전부 다운받는다.
- 이 때 이 파일들의 용량이 장난 아니게 크기 때문에 symbolic link로 goinfre에 연결시켜주어야 한다.
- minikube는 $HOME/.minikube 경로를 사용한다.
- 따라서 다음 명령어를 차례대로 입력해주자.

```cpp
cd ~
cd goinfre
mkdir .minikube
cd ..
ln -s goinfre/.minikube .minikube
```

- 참고로, 42 Seoul의 Cluster에서는 goinfre를 제외한 홈 디렉토리의 모든 파일이 자리를 옮겨도 유지가 된다.
- 따라서 이 작업은 최초 한번만 해주고, 만약 자리를 옮길 경우 다음과 같은작업을 해주어야 한다.

```cpp
mkdir ~/goinfre/.minikube

만약 같은 자리에서 다시 켰다면 이 작업은 해줄 필요가 없다.
그 이유는 goinfre의 내용은 해당 자리에 계속 남아있기 때문이다.
```

- 그 후, minikube를 실행시켜 준다.
- `minikube start --driver=virtualbox`
- 조금 기다리면 파일을 내려받으면서 Done! 이라는 메세지와 함께 실행될 것이다.
- 

[minikube를 사용하면서 자주 쓰는 명령어들](./command/README.md)
