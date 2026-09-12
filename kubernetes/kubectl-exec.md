# 06 kubectl exec

## 1. Pod 내부 명령 실행

```bash
kubectl exec -it nginx -- bash
```
사용자가 Pod에 직접 SSH하는 것이 아니다.

Kubernetes API를 통해 실행 요청이 전달된다.

## 2. 전체 흐름
```text
kubectl
   ↓
API Server
   ↓ TCP 10250
kubelet
   ↓
containerd
   ↓
nginx container
```
## 3. API Server → kubelet 확인

worker2에서:

sudo tcpdump -ni ens3 \
  host 10.86.202.17 and tcp port 10250

확인되는 형태:

10.86.202.17 → 10.86.202.137:10250

여기서:

10.86.202.17
→ master Node

10.86.202.137
→ worker2 Node

10250
→ kubelet 통신

이다.

## 4. containerd 확인
```bash
sudo crictl ps
```
Node에서 containerd가 관리하는 Container를 확인할 수 있다.

kubelet과 containerd의 통신은 일반적인 TCP 네트워크가 아니라
Unix Domain Socket을 사용한다.
```text
kubelet
   ↓
/run/containerd/containerd.sock
   ↓
containerd
```
## 5. exec와 Pod Network는 별개

예를 들어:
```bash
kubectl exec nginx -- curl http://10.0.2.223
```
라고 실행하면 두 종류의 통신이 발생한다.

명령 전달
```text
kubectl
 ↓
API Server
 ↓ TCP 10250
kubelet
 ↓
containerd
 ↓
nginx
curl이 발생시키는 실제 네트워크
10.0.0.42
 ↓
Cilium
 ↓
VXLAN
 ↓
10.0.2.223
```
따라서 Pod 안에서 실행한 명령을 전달하는 Kubernetes Control Path와
그 명령이 발생시키는 Pod Data Path는 서로 다른 경로다.
