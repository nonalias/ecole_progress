# MetalLB

- Kubernetes 에서는 기본적으로 로드밸런서를 제공하지 않는다.
- 따라서 클라우드 환경이 아닌 일반 로컬 환경에서는 LoadBalancer 기능을 사용하기 위해 플러그인을 설치해주어야 한다.
- 그 과정은 다음과 같다. (참고 : [https://metallb.universe.tf/installation/](https://metallb.universe.tf/installation/))

```cpp
mkdir metalLB
cd metalLB

# 링크를 바로 apply 해도 되지만 링크가 바뀔 수 있으므로 다운로드받아 사용한다.
# namespace 생성
curl https://raw.githubusercontent.com/metallb/metallb/v0.9.4/manifests/namespace.yaml -o namespace.yaml
kubectl apply -f namespace.yaml
# metallb 설정
curl https://raw.githubusercontent.com/metallb/metallb/v0.9.4/manifests/metallb.yaml -o metallb.yaml
kubectl apply -f metallb.yaml

# secret key 생성
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"

# ConfigMap 설정
vim config.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.99.240 - 192.168.99.250

kubectl apply -f config.yaml
```
