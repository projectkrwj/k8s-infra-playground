# 03. Kubespray Setup

## 1. Kubespray 이동

```bash
cd ~/kubespray
```
## 2. Kubespray 버전 확인
```bash
git describe --tags --always
```
실습 당시:

v2.31.0-126-g8e8751465

Kubespray를 이용하여 Kubernetes Cluster를 구성한다.

## 3. Python / Ansible 환경 확인
```bash
python --version
ansible --version
pip check
```
Kubespray 전용 Python virtual environment에서 Ansible을 실행했다.

주요 환경:

Python       3.11.0rc1
Ansible      11.13.0
ansible-core 2.18.19

## 4. Inventory 확인
```bash
ansible-inventory \
  -i inventory/mycluster/hosts.yaml \
  --graph
```
클러스터 구성:
```text
etcd
└── master

kube_control_plane
└── master

kube_node
├── worker1
└── worker2
```
즉:
```text
master
 └─ Control Plane + etcd

worker1
 └─ Worker

worker2
 └─ Worker
 ```
## 5. Kubernetes 설치
```bash
ansible-playbook \
  -i inventory/mycluster/hosts.yaml \
  --become --become-user=root \
  cluster.yml
```
Kubespray가 각 Node에 Kubernetes 구성 요소를 설치한다.

주요 구성:
```text
Control Plane
├─ kube-apiserver
├─ kube-controller-manager
├─ kube-scheduler
└─ etcd

Worker
├─ kubelet
├─ containerd
└─ kube-proxy
```
## 6. Node 상태 확인
```bash
kubectl get nodes
```
초기 CNI 설정에 문제가 있을 경우:

NotReady

상태가 나타날 수 있다.

## 7. Node 상세 상태 확인
```bash
kubectl describe node worker1
```
다음과 같은 오류를 확인했다.

NetworkPluginNotReady
cni plugin not initialized

이는 Kubernetes Node 자체가 생성되지 않은 것이 아니라
kubelet이 사용할 CNI가 정상적으로 초기화되지 않았다는 의미다.

## 8. CNI 설정 확인
```bash
ls /etc/cni/net.d/
```
초기에는 CNI 설정 파일이 존재하지 않았다.

또한:
```bash
ls /opt/cni/bin/
```
을 통해 CNI Plugin Binary가 설치되어 있는지 확인할 수 있다.

## 9. CNI 문제 해결 방향

Kubespray에서 CNI를 직접 설치하는 대신
Kubernetes Cluster를 먼저 구성한 후 Cilium을 별도로 설치하는 방식으로 변경했다.

설정:

kube_network_plugin: cni
