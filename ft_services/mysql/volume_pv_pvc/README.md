# Volume 그리고 PVC, PV

## Volume

- 쿠버네티스에서 볼륨이란, Pod에 종속되는 디스크이다.
- Pod 단위이기 때문에 그 Pod에 속해있는 하나 이상의 컨테이너들은 모두 이 볼륨을 공유해서 사용할 수 있다.

### 볼륨 종류

- Temp
    - emptyDir
        - emptyDir은 Pod가 생성될 때 생성되고, Pod가 삭제될 때 같이 삭제되는 임시볼륨이다.
        - 단, Pod 내의 컨테이너가 crash되어 삭제되거나 재시작 되더라도 emptyDir의 생명주기는 Pod 단위이기 때문에 emptyDir은 삭제되지 않고 계속해서 사용이 가능하다.
        - 생성 당시에는 디스크에 아무 내용이 없기 때문에 emptyDir이라고 한다.
        - 다음은 하나의 Pod에 nginx와 redis를 가동시키고, emptyDir볼륨을 생성하여 이를 공유하는 설정이다.

        ```jsx
        apiVersion: v1
        kind: Pod
        metadata:
          name: shared-volumes
        spec:
          containers:
          - name: redis
            image: redis
            volumeMounts:
            - name: shared-storage
        			# shared-storage는 emptyDir의 이름이며, 
        			# 팟 내의 어떤 Container에서 접근하더라도 이 이름가지고 접근이 가능하다.
        			# mountPath는 그 Container에서 경로를 어떻게 할지 정해준다. (자율)
              mountPath: /data/shared
          - name: nginx
            image: nginx
            volumeMounts:
            - name: shared-storage
              mountPath: /data/shared
          volumes:
          - name: shared-storage
            emptyDir: {}
        ```

        - 위의 내용으로 파일(new.yaml)을 만들어 주고, kubectl apply -f new.yaml을 실행.

        ```jsx
        그리고나서 

        kubectl exec -it pods/shared-volumes --container redis -- /bin/bash

        를 입력해서 redis container에 들어가자.
        ```

        ![Volume%20%E1%84%80%E1%85%B3%E1%84%85%E1%85%B5%E1%84%80%E1%85%A9%20PVC,%20PV%2018a99d26f0f84367a4181bedcc90e30d/Untitled.png](Volume%20그리고%20PVC%2C%20PV%2018a99d26f0f84367a4181bedcc90e30d/Untitled.png)

        - 당연하게도, 아무 파일도 넣지 않았기 때문에 아무것도 없다. 여기서

        ```jsx
        touch hello
        ```

        - 를 입력하여 새로운 파일을 만들어준다. 그리고 나서  다음 명령어를 입력해서 nginx 컨테이너에 들어가보자.

        ```jsx
        kubectl exec -it pods/shared-volumes --container nginx -- /bin/bash
        cd data/shared
        ls
        ```

        ![Volume%20%E1%84%80%E1%85%B3%E1%84%85%E1%85%B5%E1%84%80%E1%85%A9%20PVC,%20PV%2018a99d26f0f84367a4181bedcc90e30d/Untitled%201.png](Volume%20그리고%20PVC%2C%20PV%2018a99d26f0f84367a4181bedcc90e30d/Untitled%201.png)

- Local
    - hostPath
        - 노드의 로컬디스크의 경로를 Pod에서 마운트(연결)해서 사용한다.
        - 같은 hostPath에 있는 Volume은 여러 Pod 사이에서 공유되어 사용된다.
        - 또한, Pod가 삭제되더라도 hostPath에 있는 파일들은 삭제되지 않고, 다른 Pod가 같은 hostPath를 마운트하게 되면, 남아있는 파일을 엑세스 할 수 있다.

        ![Volume%20%E1%84%80%E1%85%B3%E1%84%85%E1%85%B5%E1%84%80%E1%85%A9%20PVC,%20PV%2018a99d26f0f84367a4181bedcc90e30d/Untitled%202.png](Volume%20그리고%20PVC%2C%20PV%2018a99d26f0f84367a4181bedcc90e30d/Untitled%202.png)

        - 주의할 점은, Pod가 재시작되어 **다른** 노드에서 기동될 경우, 그 노드의 hostPath를 사용하기 때문에 이전에 다른 노드에서 사용한 hostPath의 파일 내용은 엑세스가 불가능하다.
            - Pod가 삭제됬을 때 그 Node에서 재생성 되리라는 법은 없기 때문이다.
        - hostPath 예제. (/tmp디렉ㅌ토리를 hostPath를 이용하여 /data/shared 디렉토리에 마운트하여 사용하는 예제이다.)

            ```jsx
             apiVersion: v1
            kind: Pod
            metadata:
              name: hostpath
            spec:
              containers:
              - name: redis
                image: redis
                volumeMounts:
                - name: terrypath
                  mountPath: /data/shared
              volumes:
              - name: terrypath
                hostPath:
                  path: /tmp
                  type: Directory
            ```

            - k apply -f new.yaml을 입력한 후
            - Node에 접근하여 /tmp 경로에 hello.txt를 만들면 (어떻게 들어가는 지 모르겠음..)
            - kubectl exec -it pods/hostpath —container redis — /bin/bash 명령어로 redis에 들어가서 /data/shared 경로로 들어가면 파일(hello.txt)이 생성된 것을 확인할 수 있다.
- Network
    - GlusterFS, gitRepo, NFS, iSCSI, gcePersistentDisk 등
    - gitRepo
        - 생성시에 지정된 git repo의 특정 리비전(커밋?)의 내용을 clone해서 내려 받은 후에 디스크 볼륨을 생성한다.
        - 물리적으로는 emptyDir이 생성되고, (즉, Pod단위로 생명주기) git 레파지토리의 내용을 clone으로 다운 받는다.

        ```jsx
        apiVersion: v1
        kind: Pod
        metadata:
        	name: gitrepo-volume-pod
        spec:
        	containers:
        	- image: nginx-alpine
        		name: web-server
        		volumeMounts:
        		- name: html
        			mountPath: /usr/share/nginx/html
        			readOnly: true
        		ports:
        		- containerPort: 80
        			protocol: TCP
        	volumes:
        	- name: html
        		gitRepo:
        			repository: <repo-address>
        			revision: master
        			directory: .
        ```

    - PV and PVC
        - Pod 에 영속성 있는 볼륨을 제공하기 위한 환경
        - PV : PersistentVolume
        - PVC: PersistentVolumeClaim
        - 일반적으로 디스크 볼륨을 설정하기 위해서는 물리적 디스크를 생성해야 하고, 생성된 물리적 디스크에 대한 설정을 자세하게 알고있어야 한다.
        - k8s는 인프라에 대한 복잡성을 "추상화"를 통해서 간단하게 하고, 개발자들이 손 쉽게 필요한 인트라(container, disk, network)를 설정할 수 있도록 하는 개념을 가지고 있다.
        - 따라서, 인프라에 종속적인 부분은 sys admin이 설정하도록 하고, 개발자는 이에 대한 이해없이 간단하게 사용할 수 있도록 디스크 볼륨 부분에 PVC와 PV라는 개념을 도입하였다.
        - 과정
            - sys admin이 실제 물리디스크를 생성한 후에, 이 디스크를 PersistentVolume이라는 이름으로 k8s에 등록한다.
            - 개발자는 Pod를 생성할 때, 볼륨을 정의하고 이 볼륨 정의 부분에 물리적 디스크에대한 특성을 정의하는 것이 아닌, PVC를 지정하여 관리자가 생성한 PV와 연결한다.
            - 즉, 개발자는 PVC에 대한 설정만 해주면 된다.
            - sys admin은 PV를 설정해준다.

                ![Volume%20%E1%84%80%E1%85%B3%E1%84%85%E1%85%B5%E1%84%80%E1%85%A9%20PVC,%20PV%2018a99d26f0f84367a4181bedcc90e30d/Untitled%203.png](Volume%20그리고%20PVC%2C%20PV%2018a99d26f0f84367a4181bedcc90e30d/Untitled%203.png)

        - ClusterIP로 만들경우 다른 Pod에서 접근이 가능하다.

        ![Volume%20%E1%84%80%E1%85%B3%E1%84%85%E1%85%B5%E1%84%80%E1%85%A9%20PVC,%20PV%2018a99d26f0f84367a4181bedcc90e30d/Untitled%204.png](Volume%20그리고%20PVC%2C%20PV%2018a99d26f0f84367a4181bedcc90e30d/Untitled%204.png)

## 실습

- emptyDir

    `vim emptydir.yaml`

    ```jsx
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-volume-1
    spec:
      containers:
        - name: container1
          image: tmkube/init
          volumeMounts:
          - name: empty-dir
            mountPath: /mount1
        - name: container2
          image: tmkube/init
          volumeMounts:
          - name: empty-dir
            mountPath: /mount2
      volumes:
        - name: empty-dir
          emptyDir: {}
    ```

    - 적용하기 위해 다음 명령어를 입력한다.

    `kubectl apply -f emptydir.yaml`

    - 그 다음 컨테이너에 들어가 본다.

        `kubectl exec -it pod-volume-1 --container=container1 sh`

        다른 터미널 하나를 띄워 다른 컨테이너도 접속해놓자.

        `kubectl exec -it pod-volume-1 --container=container2 sh`

        - container1에서 `touch container1`를 하면

            ![Volume%20%E1%84%80%E1%85%B3%E1%84%85%E1%85%B5%E1%84%80%E1%85%A9%20PVC,%20PV%2018a99d26f0f84367a4181bedcc90e30d/Untitled%205.png](Volume%20그리고%20PVC%2C%20PV%2018a99d26f0f84367a4181bedcc90e30d/Untitled%205.png)

        - container2에서 확인할 수 있다.

            ![Volume%20%E1%84%80%E1%85%B3%E1%84%85%E1%85%B5%E1%84%80%E1%85%A9%20PVC,%20PV%2018a99d26f0f84367a4181bedcc90e30d/Untitled%206.png](Volume%20그리고%20PVC%2C%20PV%2018a99d26f0f84367a4181bedcc90e30d/Untitled%206.png)

    - 실습을 마쳤다면 Pod을 삭제해 준다.

    `kubectl delete -f emptydir.yaml`

- hostPath
    - yaml 파일을 작성해보자.

    `vim hostpath.yaml`

    ```jsx
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-volume-2
    spec:
      nodeSelector:
        kubernetes.io/hostname: minikube
      containers:
        - name: container
          image: tmkube/init
          volumeMounts:
            - name: host-path
              mountPath: /mount1
      volumes:
        - name: host-path
          hostPath:
            path: /node-v
            type: DirectoryOrCreate
    ```

    `vim hostpath2.yaml`

    ```jsx
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-volume-3
    spec:
      nodeSelector:
        kubernetes.io/hostname: minikube
      containers:
        - name: container
          image: tmkube/init
          volumeMounts:
            - name: host-path
              mountPath: /mount2
      volumes:
        - name: host-path
          hostPath:
            path: /node-v
            type: DirectoryOrCreate
    ```

    - 두 개의 파드는 각각 minikube 노드의 hostPath로 연결되어 있다.
    - 따라서, 이번에는 각 파드별로 mount한 경로로 파일이 서로 공유되는지 확인해 보자.

    `kubectl apply -f hostpath.yaml`

    `kubectl apply -f hostpath2.yaml`

    `kubectl exec -it pods/pod-volume-2 sh`

    - 그리고 다른 터미널을 열어서 다음 명령어를 입력한다.

    `kubectl exec -it pods/pod-volume-3 sh`

    - 이제, 확인해보자.

        ![Volume%20%E1%84%80%E1%85%B3%E1%84%85%E1%85%B5%E1%84%80%E1%85%A9%20PVC,%20PV%2018a99d26f0f84367a4181bedcc90e30d/Untitled%207.png](Volume%20그리고%20PVC%2C%20PV%2018a99d26f0f84367a4181bedcc90e30d/Untitled%207.png)

        ![Volume%20%E1%84%80%E1%85%B3%E1%84%85%E1%85%B5%E1%84%80%E1%85%A9%20PVC,%20PV%2018a99d26f0f84367a4181bedcc90e30d/Untitled%208.png](Volume%20그리고%20PVC%2C%20PV%2018a99d26f0f84367a4181bedcc90e30d/Untitled%208.png)

    - 미리 만들어놓은 파일들이 같이 공유되어 있는 것을 볼 수 있다.
    - 삭제는 다음과 같이 진행한다.
        - `kubectl delete -f hostpath.yaml`
        - `kubectl delete -f hostpath2.yaml`
- PV / PVC
    - 우선 yaml 파일을 작성하자.
    - `vim pv.yaml`

        ```jsx
        apiVersion: v1
        kind: PersistentVolume
        metadata:
          name: pv-01
        spec:
          capacity:
            storage: 1G
          accessModes:
            - ReadWriteOnce
          local:
            path: /tmp/
          nodeAffinity:
            required:
              nodeSelectorTerms:
                - matchExpressions:
                  - {key: kubernetes.io/hostname, operator: In, values: [minikube]}
        ```

    - 그 다음, pvc를 작성하자.
    - `vim pvc.yaml`

        ```jsx
        apiVersion: v1
        kind: PersistentVolumeClaim
        metadata:
          name: pvc-01
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1G
          storageClassName: ""
        ```

    - 이 때 pvc에서 명세한 storage와 accessMode에 따라 자동으로 PV를 선택해 준다.

        ![Volume%20%E1%84%80%E1%85%B3%E1%84%85%E1%85%B5%E1%84%80%E1%85%A9%20PVC,%20PV%2018a99d26f0f84367a4181bedcc90e30d/Untitled%209.png](Volume%20그리고%20PVC%2C%20PV%2018a99d26f0f84367a4181bedcc90e30d/Untitled%209.png)

    - 그리고 Pod 하나를 만들자.
    - `vim pod.yaml`

        ```jsx
        apiVersion: v1
        kind: Pod
        metadata:
          name: pod-volume-4
        spec:
          containers:
            - name: container
              image: nginx
              volumeMounts:
                - name: pvc-pv
                  mountPath: /mount3
          volumes:
            - name: pvc-pv
              persistentVolumeClaim:
                claimName: pvc-01
        ```

    - 여기서 pvc를 명세를 해주면, 그 pvc가 가리키는 pv와 연결이 된다.
    - 이 때 mountPath는 해당 Pod에서 pv로 mount할 수 있는 경로를 적어준다.
    - pv 가 어느 pvc에 binding이 된다면, 다른 pvc에서는 이 pv에 접근할 수 없다.
        - 즉, pv와 pvc는 일대일 매칭이다.
        - 용량이 달라도 매칭은 해준다. (적어야 할 듯 ?)
    - pv 먼저 생성, pvc 후에 연결
    -
