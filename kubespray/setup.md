# 03. Kubespray Setup

## 1. Kubespray 설치

```bash
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
```
## 2. Kubespray 버전 확인
```bash
git describe --tags --always
```
실습 당시:

v2.31.0-126-g8e8751465

Kubespray를 이용하여 Kubernetes Cluster를 구성한다.

## 3. Python / Ansible 환경 확인
우선 가상환경을 만들고 
```bash
python3 -m venv .venv
source .venv/bin/activate

python --version
ansible --version
pip check
```
Kubespray 전용 Python virtual environment에서 Ansible을 실행했다.

주요 환경:

Python       3.11.0rc1
Ansible      11.13.0
ansible-core 2.18.19

```bash
pip install -r requirements.txt
```
여기까지가 Kubespray 실행에 필요한 python패키지를 .venv 안에 설치하는 과정.

## 4. Inventory 만들기
```bash
cp -rfp inventory/sample inventory/mycluster
```
그러면 
```text
kubespray/
└── inventory/
    ├── sample/
    └── mycluster/    ← 우리가 사용할 클러스터 설정
```
이렇게 복사된다.
어떤 VM이 Master이고 어떤 VM이 worker인지 Kubespray에 알려줘야 한다.

잠시 ssh설정에 다녀오자. Kubespray가 실제로 worker들에 들어갈 수 있는지 확인하기 위해
필요한 단계이다. k8s-infra-playground/ssh/setup.md에 다녀오자.

inventory.ini 작성을 위해
```bash
nano inventory/mycluster/inventory.ini
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
```bash
[all]
master ansible_host=10.86.202.17
worker1 ansible_host=10.86.202.84
worker2 ansible_host=10.86.202.137

[kube_control_plane]
master

[etcd]
master

[kube_node]
worker1
worker2

[k8s_cluster:children]
kube_control_plane
kube_node
```
이렇게 복사해서 붙여넣는다. 본 k8s-infra-playground/ssh/setup.md 에서 본듯이
똑같이 나오면 된다.

## 5. Kubernetes 설치
```bash
ansible-playbook \
  -i inventory/mycluster/hosts.yaml \
  --become --become-user=root \
  cluster.yml
```
fail없이 다 끝난다면 cilium이 아닌 calico가 설치되어있을 것이다.
내 작업에서는 calico CNI구성이 fail되고 나머지 구성은 다 괜찮게 되었다.


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
Cilium을 별도로 설치하는 방식으로 변경했다.
