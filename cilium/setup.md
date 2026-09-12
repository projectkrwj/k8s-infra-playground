# 04. Cilium Setup

## 1. Cilium 설치

```bash
curl -L --remote-name https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
tar xzvf cilium-linux-amd64.tar.gz
sudo mv cilium /usr/local/bin

cilium install
```
실습 당시 설치된 버전:

Cilium 1.20.1

Cilium은 Kubernetes의 CNI 역할과
Pod Network Datapath를 담당한다.

## 2. Cilium 상태 확인
```bash
cilium status --wait
```
정상 상태:

Cilium: OK
Operator: OK
Envoy DaemonSet: OK

Cilium Agent는 각 Node에 DaemonSet 형태로 실행된다.

master   → cilium
worker1  → cilium
worker2  → cilium

## 3. Cilium Pod 확인
```bash
kubectl -n kube-system get pods -o wide
```
Cilium Agent가 각 Node에서 실행되고 있는지 확인한다.

## 4. Cilium Node 확인
```bash
kubectl get ciliumnodes.cilium.io -o wide
```
예:

NAME      CILIUMINTERNALIP   INTERNALIP
master    10.0.1.109         10.86.202.17
worker1   10.0.2.217         10.86.202.84
worker2   10.0.0.177         10.86.202.137

여기서:

INTERNALIP
→ Kubernetes Node가 사용하는 VM IP

CILIUMINTERNALIP
→ Cilium datapath에서 사용하는 주소

## 5. Cilium 상태에서 Datapath 확인
```bash
cilium status
```
주요 출력:

Routing: Network: Tunnel [vxlan]
Attach Mode: Legacy TC
Device Mode: veth

현재 실습 환경의 핵심 구조는:
```text
Pod
 ↓
veth
 ↓
Cilium
 ↓
VXLAN
 ↓
Node
```
이다.

## 6. CNI 설정 파일 확인
```bash
ls /etc/cni/net.d/
```
Cilium 설치 후:

05-cilium.conflist

등의 CNI 설정 파일이 생성된다.

Cilium이 Kubernetes Pod 생성 과정에 CNI로 연결된 것을 확인할 수 있다.

## 7. Cilium IP 확인

Cilium Agent 내부에서:
```bash
kubectl -n kube-system exec ds/cilium -- \
  cilium-dbg ip list
```
Cilium이 관리하는 Endpoint / IP 정보를 확인할 수 있다.

Kubernetes API를 이용하면:
```bash
kubectl get ciliumendpoints -A -o wide
```
또는:
```bash
kubectl get ciliumnodes.cilium.io -o wide
```
로 확인할 수 있다.
