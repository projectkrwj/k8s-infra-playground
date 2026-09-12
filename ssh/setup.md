# 02. SSH Setup

Kubespray는 Ansible을 이용해 여러 Node에 원격으로 명령을 실행한다.

따라서 Kubernetes를 배포하기 전에
관리 Node에서 다른 Node로 SSH 접속이 가능하도록 구성한다.

---

## 1. SSH 접속 확인

현재는 master의 public key가 worker1, 2 각각에 저장이 되어 있지 않기 때문에
master에서 worker1로 접속하는 것이 불가능하다. 적어도 password를 입력해야 한다.

```bash
ssh worker1
```
worker2:
```bash
ssh worker2
```
이렇게 접근할 수 있어야 한다.


## 2. SSH Key 생성

master에서 SSH Key를 생성한다.
```bash
ssh-keygen -t ed25519
```
기본 경로:
```bash
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```
요놈들이 생겼는지 확인한 뒤에
Private Key는 master에 보관하고,
Public Key를 접속 대상 Node에 등록한다.

## 3. Public Key 전달
```bash
multipass exec worker1 -- bash -c "echo '여기에_복사한_공개키' >> /home/ubuntu/.ssh/authorized_keys"
multipass exec worker2 -- bash -c "echo '여기에_복사한_공개키' >> /home/ubuntu/.ssh/authorized_keys"
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```
각 worker의 ~/.ssh/authorized_keys에
master의 Public Key가 등록된다.
또한 현재는 master에서 master에 ssh접근이 불가한 상태이므로 스스로의 pub키를 저장해줘야 한다.

## 4. pong되는지 확인
```bash
ansible all -i inventory/mycluster/inventory.ini -m ping
```
또한 ssh연결을 시도했을 때
비밀번호를 추가로 입력하지 않고 접속할 수 있으면
SSH Key 기반 인증이 정상적으로 구성된 것이다.

## 5. Ansible로 노드 기본 상태 확인
```bash
ansible-inventory -i inventory/mycluster/inventory.ini --graph
```
이렇게 하면 inventory가 Kubespray가 기대하는 구조로 들어갔는지 확인 가능.
```text
@all:
 |--@ungrouped:
 |--@etcd:
 |    |--master
 |--@k8s_cluster:
 |    |--@kube_control_plane:
 |    |    |--master
 |    |--@kube_node:
 |    |    |--worker1
 |    |    |--worker2
```
이 출력이면 inventory 구조가 의도한 대로 들어간 것.


## 6. SSH 연결 확인

master에서:
```bash
ssh worker1 hostname
ssh worker2 hostname
```
예상 결과:

worker1
worker2

즉 master에서 각 worker로
원격 명령을 직접 실행할 수 있다.

## 7. Kubespray와 SSH의 관계

전체 배포 과정은 대략 다음과 같다.
```text
master
  │
  │ SSH
  ├──────────────→ worker1
  │
  └──────────────→ worker2
          │
          ▼
       Ansible
          │
          ▼
      Kubespray
          │
          ▼
 Kubernetes 설치
```
즉 Kubespray 자체가 Node에 직접 설치되는 것이 아니라,
master에서 실행한 Ansible이 SSH를 통해 각 Node에 접속하여
Kubernetes 구성 작업을 수행한다.

이제 다시 k8s/kubespray/setup.md로 돌아가자
