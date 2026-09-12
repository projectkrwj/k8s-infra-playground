# k8s-infra-playground

Multipass와 Kubespray를 이용해 Kubernetes 클러스터를 구축하고,
Cilium 기반의 Pod Network와 VXLAN Overlay Network의 동작을
실제 패킷 캡처를 통해 확인하는 실습 프로젝트.

단순한 Kubernetes 설치에 그치지 않고, Kubernetes의 Control Plane과 Data Plane이 실제 네트워크에서 어떻게 동작하는지 직접 확인하는 것을 목표로 한다.

---

## 1. Environment

* Windows
* WSL2 / Ubuntu
* Multipass
* Kubernetes v1.36.4
* Kubespray v2.31.0
* containerd
* Cilium v1.20.1
* tcpdump

### Cluster

| Node    | Node IP         | Role          |
| ------- | --------------- | ------------- |
| master  | `10.86.202.17`  | Control Plane |
| worker1 | `10.86.202.84`  | Worker        |
| worker2 | `10.86.202.137` | Worker        |

---

## 2. Kubernetes Architecture

기본적인 Kubernetes 실행 흐름을 다음과 같이 정리했다.

```text
kubectl
   │
   ▼
API Server
   │
   ▼
kubelet
   │
   ▼
containerd
   │
   ▼
Pod / Container
```

각 구성요소의 역할:

* **kubectl**: Kubernetes API를 호출하는 CLI
* **API Server**: Kubernetes의 중앙 API endpoint
* **kubelet**: 각 Node에서 Pod의 상태와 실행을 관리
* **containerd**: 컨테이너 실행을 담당하는 Container Runtime
* **Pod**: Kubernetes에서 애플리케이션이 실행되는 기본 단위

---

## 3. Kubespray

Multipass로 생성한 VM들을 대상으로 Kubespray를 이용해 Kubernetes Cluster를 구성했다.

```text
master
 ├── etcd
 ├── kube-apiserver
 ├── kube-controller-manager
 └── kube-scheduler

worker1
 └── kubelet
     └── containerd

worker2
 └── kubelet
     └── containerd
```

### CNI 구성 과정

처음에는 Kubespray를 통해 CNI까지 구성하려 했으나 CNI 초기화 과정에서 다음과 같은 문제가 발생했다.

```text
NetworkPluginNotReady
Network plugin returns error:
cni plugin not initialized
```

이후 CNI 구성과 Kubernetes Node 상태의 관계를 직접 확인하고, Cilium을 별도로 설치하는 방식으로 실습을 진행했다.

---

## 4. Cilium

Cilium v1.20.1을 설치한 후 모든 Node가 `Ready` 상태가 되는 것을 확인했다.

```text
Cilium: OK
Operator: OK
DaemonSet cilium: 3/3 Ready
```

Cilium은 Kubernetes Pod Network의 datapath를 구성하며, 현재 환경에서는 VXLAN 기반의 Overlay Network를 사용한다.

확인된 Cilium 상태:

```text
Routing: Network: Tunnel [vxlan]
Device Mode: veth
Attach Mode: Legacy TC
Masquerading: IPTables
```

---

## 5. Pod Network

서로 다른 Worker Node에 Pod를 배치하여 Pod-to-Pod 통신을 테스트했다.

```text
worker2                         worker1

Pod                            Pod
10.0.0.42                      10.0.2.223
   │                               │
   └──────── Pod Network ──────────┘
```

Pod IP와 Node IP는 서로 다른 Network 영역에 존재한다.

```text
Node Network

10.86.202.0/24
        │
        ├── master   10.86.202.17
        ├── worker1  10.86.202.84
        └── worker2  10.86.202.137


Pod Network

10.0.x.x
        │
        ├── worker2 Pod  10.0.0.42
        └── worker1 Pod  10.0.2.223
```

---

## 6. veth

Pod는 별도의 Linux Network Namespace를 사용한다.

Pod Network Namespace와 Node Network Namespace 사이에는 veth pair가 연결된다.

```text
Pod Network Namespace
┌─────────────────────┐
│                     │
│       eth0          │
│        │            │
└────────┼────────────┘
         │
       veth pair
         │
┌────────┼────────────┐
│        │            │
│   Cilium / Host     │
│                     │
└─────────────────────┘
```

따라서 Pod의 `eth0`와 Node에서 보이는 `lxc...` 계열 인터페이스는 가상 Ethernet link를 구성한다.

---

## 7. VXLAN Overlay Network

서로 다른 Node에 존재하는 Pod끼리 통신할 때 Cilium은 VXLAN을 이용해 Pod packet을 encapsulation한다.

### Pod 관점

```text
10.0.0.42 → 10.0.2.223
```

하지만 실제 Node Network에서는:

```text
10.86.202.137 → 10.86.202.84
UDP 8472
```

로 전달된다.

개념적으로:

```text
┌─────────────────────────────────────────────┐
│ Outer Packet                                │
│                                             │
│ 10.86.202.137 → 10.86.202.84               │
│ UDP 8472                                    │
│                                             │
│   ┌─────────────────────────────────────┐   │
│   │ Inner Packet                        │   │
│   │                                     │   │
│   │ 10.0.0.42:33070 → 10.0.2.223:80    │   │
│   │                                     │   │
│   │ TCP / HTTP                          │   │
│   └─────────────────────────────────────┘   │
│                                             │
└─────────────────────────────────────────────┘
```

즉 Pod Network의 주소를 그대로 유지하면서 Node Network를 통해 다른 Node까지 전달할 수 있다.

---

## 8. VXLAN Packet Capture

worker2에서 실제 VXLAN packet을 tcpdump로 확인했다.

```bash
sudo tcpdump -ni ens3 udp port 8472
```

관찰한 packet:

```text
10.86.202.137.38940 > 10.86.202.84.8472

IP 10.0.0.42.33070 > 10.0.2.223.80:
Flags [S]
```

이를 통해 하나의 packet 안에서 다음 두 Network layer를 확인할 수 있었다.

```text
Outer:

10.86.202.137 → 10.86.202.84
UDP 8472


Inner:

10.0.0.42 → 10.0.2.223
TCP 80
```

Pod Network packet이 VXLAN으로 encapsulation되어 실제 Node Network를 통해 전달되는 것을 직접 확인했다.

---

## 9. Pod-to-Pod HTTP Communication

worker2의 Pod에서 worker1의 Pod로 HTTP request를 발생시켰다.

```text
10.0.0.42:33070 → 10.0.2.223:80
```

TCP 3-way handshake:

```text
10.0.0.42 → 10.0.2.223    SYN
10.0.2.223 → 10.0.0.42    SYN/ACK
10.0.0.42 → 10.0.2.223    ACK
```

이후:

```text
HTTP GET /
        ↓
HTTP/1.1 200 OK
```

가 정상적으로 관찰되었다.

`cilium_vxlan`에서 확인하면:

```text
10.0.0.42 → 10.0.2.223
```

이고,

`ens3`에서 확인하면:

```text
10.86.202.137 → 10.86.202.84
UDP 8472
```

로 나타난다.

---

## 10. kubectl exec

`kubectl exec`의 동작도 tcpdump를 통해 확인했다.

다음 명령을 실행하면:

```bash
kubectl exec nginx -- ...
```

대략적인 실행 흐름은 다음과 같다.

```text
kubectl
   │
   ▼
API Server
   │
   │ HTTPS / TCP 10250
   ▼
worker2 kubelet
   │
   ▼
containerd
   │
   ▼
nginx Pod
```

여기서 중요한 점은 `kubectl exec`의 제어 경로와 Pod-to-Pod 데이터 경로가 서로 다르다는 것이다.

---

## 11. API Server → kubelet Packet Capture

worker2에서 다음 명령을 실행하여 API Server와 kubelet 사이의 통신을 관찰했다.

```bash
sudo tcpdump -ni ens3 \
  host 10.86.202.17 and tcp port 10250
```

실제 관찰 결과:

```text
10.86.202.17.33448 > 10.86.202.137.10250: Flags [S]
10.86.202.137.10250 > 10.86.202.17.33448: Flags [S.]
10.86.202.17.33448 > 10.86.202.137.10250: Flags [.]
```

이는 TCP 3-way handshake를 나타낸다.

```text
master                         worker2
10.86.202.17                   10.86.202.137
     │                              │
     │────── SYN ─────────────────>│
     │<───── SYN/ACK ──────────────│
     │────── ACK ─────────────────>│
     │                              │
     │       TCP 10250              │
     │<──── kubelet connection ────>│
```

---

## 12. Control Plane vs Data Plane

이번 실습에서 Kubernetes의 두 가지 서로 다른 네트워크 흐름을 구분할 수 있었다.

### Control / Management Path

```text
kubectl
   │
   ▼
API Server
   │
   │ TCP 10250
   ▼
kubelet
   │
   ▼
containerd
   │
   ▼
Pod
```

이 통신은 Node IP를 이용한다.

```text
10.86.202.17 → 10.86.202.137
```

### Pod Data Path

```text
Pod
10.0.0.42
   │
   ▼
Cilium
   │
   │ VXLAN
   ▼
worker1
   │
   ▼
Pod
10.0.2.223
```

실제 Node Network에서는:

```text
10.86.202.137 → 10.86.202.84
UDP 8472
```

로 전달된다.

---

## 13. Key Concepts

이번 실습을 통해 다음 개념들을 직접 확인했다.

* Kubernetes Control Plane
* API Server
* kubelet
* containerd
* CRI / OCI 구조
* CNI
* Cilium
* Linux Network Namespace
* veth pair
* Pod IP / Node IP
* Overlay Network
* Underlay Network
* VXLAN Encapsulation
* TCP 3-way Handshake
* Kubernetes `kubectl exec`
* kubelet TCP 10250
* tcpdump를 이용한 packet analysis
* Kubernetes Control Plane / Data Plane

---

## 14. Next Steps

다음 단계에서는 Cilium이 실제 packet을 어떻게 처리하는지 더 깊게 확인한다.

```text
Kubernetes Network
        │
        ▼
CNI
        │
        ▼
Cilium
        │
        ▼
eBPF
        │
        ▼
TC / Network Datapath
        │
        ▼
VXLAN
```

예정된 실습:

* Cilium eBPF datapath
* Cilium endpoint
* Cilium IPAM
* Service / ClusterIP
* kube-proxy
* NetworkPolicy
* Hubble
* Pod-to-Service traffic
* eBPF 기반 packet observability

---

## Summary

이번 실습의 핵심은 Kubernetes 네트워크를 추상적인 개념으로만 이해하지 않고 실제 packet level에서 확인하는 것이었다.

```text
                 Control Plane
                      │
                      │ TCP 10250
                      ▼
                  worker2
                      │
                   kubelet
                      │
                  containerd
                      │
                      ▼
               Pod 10.0.0.42
                      │
                      │
                 Cilium
                      │
                VXLAN Tunnel
                      │
                      ▼
               worker1
                      │
                      ▼
               Pod 10.0.2.223
```

특히 `tcpdump`를 이용해 동일한 Pod-to-Pod packet이

```text
10.0.0.42 → 10.0.2.223
```

이라는 Overlay Network의 packet에서

```text
10.86.202.137 → 10.86.202.84
UDP 8472
```

라는 Underlay Network의 packet으로 encapsulation되는 과정을 직접 확인했다.
