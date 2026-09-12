# 05. Kubernetes Node and Pod

## 1. Node 확인

```bash
kubectl get nodes
```
정상적으로 구성된 경우:

master    Ready
worker1   Ready
worker2   Ready

Ready는 해당 Node의 kubelet과 필요한 Node 구성 요소가 정상적으로 동작하고 있음을 의미한다.

## 2. Pod 생성
```bash
kubectl run nginx --image=nginx
```
Kubernetes API Server에 nginx Pod 생성을 요청한다.

## 3. Pod 확인
```bash
kubectl get pods -o wide
```
예:

NAME    READY   STATUS    IP         NODE
nginx   1/1     Running   10.0.0.42  worker2

여기서 확인할 수 있는 정보:

IP
→ Pod IP

NODE
→ Pod가 실행 중인 Kubernetes Node
## 4. Pod IP만 확인
```bash
kubectl get pod nginx \
  -o jsonpath='{.status.podIP}{"\n"}'
```
결과:

10.0.0.42

Pod IP는 Node IP와 다른 주소 대역을 사용한다.

Node
10.86.202.137

Pod
10.0.0.42
## 5. Pod가 어느 Node에서 실행되는지 확인
```bash
kubectl get pod nginx -o wide
```
예:

nginx → worker2

따라서:
```text
worker2
 └─ nginx
    └─ 10.0.0.42
```
구조가 된다.

## 6. Pod Network Namespace 확인

Node에서:

ip link

Cilium이 생성한:

lxc...

형태의 인터페이스를 확인할 수 있다.

Pod의 eth0와 Node의 lxc...는
veth pair로 연결되어 있다.
```text
Pod Namespace
    │
   eth0
    │
   veth
    │
   lxc...
    │
Node Namespace
```
## 7. Pod IP가 Node의 ip addr에 없는 이유

Node에서:
```bash
ip addr
```
를 실행해도:

10.0.0.42

가 직접 보이지 않을 수 있다.

Pod는 별도의 Linux Network Namespace를 사용하기 때문이다.

즉:
```text
Node Namespace
 └─ 10.86.202.137

Pod Namespace
 └─ 10.0.0.42
```
서로 다른 Network Namespace에 존재한다.
