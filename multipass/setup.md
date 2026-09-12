# 01. Multipass VM Setup

## 1. VM 생성

```bash
multipass launch --name master --cpus 2 --memory 4G --disk 20G
multipass launch --name worker1 --cpus 2 --memory 4G --disk 20G
multipass launch --name worker2 --cpus 2 --memory 4G --disk 20G
```
Kubernetes 실습을 위해 VM 3개를 생성한다.

master
worker1
worker2

## 2. VM 상태 확인
```bash
multipass list
```
VM의 실행 상태와 IP를 확인한다.

실습 환경에서는 다음과 같이 구성했다.

master   10.86.202.17
worker1  10.86.202.84
worker2  10.86.202.137


## 3. VM 접속
```bash
multipass shell master
```
또는:
```bash
multipass shell worker1
multipass shell worker2
```
이제부터 실행하는 ip, tcpdump, crictl 등의 명령은
해당 VM 내부의 Linux 환경에서 실행된다.

## 4. Linux Network Interface 확인
ip addr

worker2에서 다음과 같은 인터페이스를 확인할 수 있다.

ens3
cilium_host
cilium_vxlan
lxc...

ens3는 VM에 제공된 네트워크 인터페이스이다.

물리적인 노트북 NIC가 직접 보이는 것이 아니라,
Multipass/Hypervisor가 VM에 제공한 가상 NIC이다.

구조:

Physical NIC
     ↓
Windows Network
     ↓
Hypervisor / Multipass
     ↓
VM ens3

VM 내부에서는 ens3를 일반적인 Linux NIC처럼 사용할 수 있다.

## 5. VM Network 확인
```bash
ip addr show ens3
```
worker2:

10.86.202.137/24

이 주소는 Kubernetes Pod IP가 아니라
VM 자체가 사용하는 Node Network 주소이다.

Node IP
10.86.202.137

Pod IP
10.0.0.x

서로 다른 네트워크를 사용한다.
