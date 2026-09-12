# Linux Network Interfaces

## 1. 전체 인터페이스 확인

```bash
ip addr
```
worker2에서 주요 인터페이스:

ens3
cilium_host
cilium_vxlan
lxc...
## 2. ens3
```bash
ip addr show ens3
```

예:

10.86.202.137/24

VM이 사용하는 Underlay Network Interface다.

Pod Network
10.0.0.x

Node Network
10.86.202.x

Node-to-Node 통신은 기본적으로 ens3를 통해 이루어진다.

## 3. cilium_host
```bash
ip addr show cilium_host
```
예:

10.0.0.177/32

Cilium이 Node 내부에서 Pod Network와 통신하기 위해 사용하는 Host-side interface다.

## 4. cilium_vxlan
```bash
ip addr show cilium_vxlan
```
Cilium이 Node 간 Pod Network를 전달하기 위해 사용하는 VXLAN interface다.
```text
worker2
    ↓
cilium_vxlan
    ↓
VXLAN
    ↓
worker1
```
## 5. lxc 인터페이스
```bash
ip link
```
다음과 같은 인터페이스를 확인할 수 있다.

lxc...

Cilium이 Pod Network Namespace와 연결하기 위해 사용하는
veth pair의 Node-side 인터페이스다.

구조:
```text
Pod Namespace
    │
    │ eth0
    │
    │ veth pair
    │
    ▼
lxc...
    │
Node Namespace
```
---


# Linux Routing

## 1. Routing Table 확인

```bash
ip route
```
worker2에서:

default via 10.86.202.1 dev ens3
10.0.0.0/24 via 10.0.0.177 dev cilium_host
10.0.1.0/24 via 10.0.0.177 dev cilium_host
10.0.2.0/24 via 10.0.0.177 dev cilium_host
10.86.202.0/24 dev ens3
## 2. Default Route
```bash
default via 10.86.202.1 dev ens3
```
특정 목적지에 더 구체적인 Route가 없을 경우
10.86.202.1 Gateway로 패킷을 전달한다.

Node
10.86.202.137
    ↓
Gateway
10.86.202.1
## 3. Node Network
```bash
10.86.202.0/24 dev ens3
```
worker2가 속한 Underlay Network다.

master   10.86.202.17
worker1  10.86.202.84
worker2  10.86.202.137

Node 간 통신은 이 네트워크를 통해 가능하다.

## 4. Pod Network Route

10.0.0.0/24 via 10.0.0.177 dev cilium_host
10.0.1.0/24 via 10.0.0.177 dev cilium_host
10.0.2.0/24 via 10.0.0.177 dev cilium_host

Cilium이 Pod Network에 대한 Route를 구성한 것이다.

예를 들어 worker2에서:

10.0.2.0/24

목적지로 패킷을 보내면:

worker2
    ↓
cilium_host
    ↓
Cilium
    ↓
VXLAN
    ↓
worker1

형태로 전달될 수 있다.

## 5. MTU 확인
```bash
ip route
```
일부 Cilium Route에:

mtu 1450

가 표시된다.

Underlay MTU가:

1500

이고 VXLAN Encapsulation에 추가 Header가 필요하기 때문에
Pod Network에서 사용할 수 있는 MTU가 감소한다.


---


# veth

## 1. veth 확인

```bash
ip link
```
Cilium 환경에서:

lxc...

형태의 인터페이스를 확인한다.

## 2. veth란?

Linux의 Virtual Ethernet Device Pair다.

두 개의 가상 Ethernet Interface가 한 쌍으로 연결된다.
```text
Interface A
    │
    │ virtual cable
    │
Interface B
```
한쪽에서 보낸 Ethernet Frame은 반대쪽으로 전달된다.

## 3. Kubernetes Pod와 veth

Pod는 별도의 Network Namespace를 사용한다.
```text
Pod Namespace
┌─────────────┐
│ eth0        │
└──────┬──────┘
       │
      veth
       │
┌──────▼──────┐
│ lxc...      │
│ Node        │
└─────────────┘
```
Pod의 eth0는 Pod Namespace에 존재하고,
반대쪽 veth interface는 Node Namespace에 존재한다.

## 4. Pod IP와 veth

Pod:

10.0.0.42

Node:

10.86.202.137

서로 다른 IP 대역이어도 문제없다.

veth 자체는 단순한 L2 연결이고,
실제 IP Routing과 Cilium Datapath가 패킷 전달을 결정한다.

## 5. Cilium에서의 역할

현재 환경:

Pod
 ↓
eth0
 ↓
veth
 ↓
lxc...
 ↓
Cilium TC/eBPF
 ↓
VXLAN 또는 다른 Datapath

따라서 veth와 eBPF, VXLAN은 같은 개념이 아니다.

veth
→ Pod와 Node를 연결

eBPF/TC
→ 패킷 처리

VXLAN
→ Node 간 Pod Network를 Tunnel로 전달

---


# VXLAN

## 1. Cilium Routing 방식 확인

```bash
cilium status
```
주요 출력:

Routing: Network: Tunnel [vxlan]

현재 Cilium은 VXLAN Tunnel Mode를 사용한다.

## 2. VXLAN Interface 확인
```bash
ip link show cilium_vxlan
```
Node에 Cilium VXLAN Interface가 생성되어 있다.

## 3. Pod-to-Pod Traffic

예를 들어:

Pod A
10.0.0.42

Pod B
10.0.2.223

가 통신한다고 가정한다.

원래 패킷:

10.0.0.42 → 10.0.2.223
TCP 80
## 4. VXLAN Encapsulation

Node가 서로 다른 경우
원래 Pod Packet을 VXLAN으로 Encapsulation한다.

Inner Packet

10.0.0.42
    ↓
10.0.2.223
TCP 80

이를:

Outer Packet

10.86.202.137
    ↓
10.86.202.84
UDP 8472

형태로 감싼다.

## 5. 전체 구조
worker2
10.86.202.137
```text
Pod
10.0.0.42
   │
   ▼
Cilium
   │
   ▼
VXLAN
   │
   │ UDP 8472
   │
   ▼
worker1
10.86.202.84
   │
   ▼
Pod
10.0.2.223
```
## 6. Packet Structure
```text
┌─────────────────────────────┐
│ Outer IP                    │
│ 10.86.202.137               │
│       ↓                     │
│ 10.86.202.84                │
├─────────────────────────────┤
│ UDP 8472                    │
├─────────────────────────────┤
│ VXLAN Header                │
├─────────────────────────────┤
│ Inner IP                    │
│ 10.0.0.42                   │
│       ↓                     │
│ 10.0.2.223                  │
├─────────────────────────────┤
│ TCP 80                      │
└─────────────────────────────┘
```
핵심은:

Outer = 실제 Node Network
Inner = Kubernetes Pod Network

이다.


---


# tcpdump

실제 패킷이 어떤 Interface를 통과하는지 확인한다.

---

## 1. Cilium VXLAN Interface에서 확인

```bash
sudo tcpdump -ni cilium_vxlan
```
확인한 패킷:

10.0.0.42.48924 > 10.0.2.223.80

이는 Pod IP 간 통신이다.

10.0.0.42
    ↓
10.0.2.223:80

TCP 3-way handshake와 HTTP Request/Response를 확인했다.

## 2. Node Interface에서 VXLAN 확인
sudo tcpdump -ni ens3 udp port 8472

확인한 패킷:

10.86.202.137.38940
>
10.86.202.84.8472

그리고 내부에는:

10.0.0.42.33070
>
10.0.2.223.80

패킷이 존재한다.

즉:

Outer:
10.86.202.137 → 10.86.202.84
UDP 8472

Inner:
10.0.0.42 → 10.0.2.223
TCP 80
## 3. VXLAN Packet 확인 결과
```
Pod
10.0.0.42
    │
    │ TCP
    ▼
Cilium
    │
    │ VXLAN Encapsulation
    ▼
worker2 ens3
10.86.202.137
    │
    │ UDP 8472
    ▼
worker1 ens3
10.86.202.84
    │
    │ VXLAN Decapsulation
    ▼
Pod
10.0.2.223
```
이를 통해 Cilium의 VXLAN Overlay Network가
실제로 Node Network 위에서 동작하는 것을 확인했다.

## 4. Kubernetes API Server → kubelet 확인

worker2에서:
```bash
sudo tcpdump -ni ens3 \
  host 10.86.202.17 and tcp port 10250
```
확인:

10.86.202.17.33448
>
10.86.202.137.10250

이는 master와 worker2 사이의 kubelet 통신이다.
```text
master
10.86.202.17
    │
    │ TCP 10250
    ▼
worker2
10.86.202.137
    │
    ▼
kubelet
```
## 5. Pod Traffic과 Control Traffic 비교
```text
Kubernetes Control Path
API Server
10.86.202.17
      │
      │ TCP 10250
      ▼
worker2
10.86.202.137
      │
      ▼
kubelet
Pod Data Path
Pod
10.0.0.42
      │
      ▼
Cilium
      │
      ▼
VXLAN
      │
      ▼
Pod
10.0.2.223
```
따라서 kubectl exec을 수행하는 과정과
Pod 내부에서 실행한 curl이 발생시키는 네트워크 패킷은
서로 다른 Network Path를 사용한다.

## 6. 최종적으로 확인한 구조
```text
                     Kubernetes Control Plane
                              │
                       TCP 10250
                              │
                              ▼
                         kubelet
                              │
                         containerd
                              │
                              ▼
┌─────────────────────────────────────────────────┐
│                    worker2                      │
│                                                 │
│   Pod 10.0.0.42                                 │
│        │                                        │
│       eth0                                      │
│        │                                        │
│       veth                                      │
│        │                                        │
│      lxc...                                     │
│        │                                        │
│    Cilium / TC / eBPF                           │
│        │                                        │
│    cilium_vxlan                                 │
│        │                                        │
│       ens3                                      │
└────────┼────────────────────────────────────────┘
         │
         │ Outer: 10.86.202.137 → 10.86.202.84
         │ UDP 8472
         ▼
┌─────────────────────────────────────────────────┐
│                    worker1                      │
│                                                 │
│                  Cilium                         │
│                     │                           │
│                     ▼                           │
│               Pod 10.0.2.223                   │
└─────────────────────────────────────────────────┘
```
