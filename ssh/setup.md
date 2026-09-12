# 02. SSH Setup

Kubespray는 Ansible을 이용해 여러 Node에 원격으로 명령을 실행한다.

따라서 Kubernetes를 배포하기 전에
관리 Node에서 다른 Node로 SSH 접속이 가능하도록 구성한다.

---

## 1. SSH 접속 확인

master에서 worker1로 접속한다.

```bash
ssh worker1
```
worker2:
```bash
ssh worker2
```
또는 IP를 직접 사용할 수 있다.
```bash
ssh ubuntu@10.86.202.84
ssh ubuntu@10.86.202.137
```
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
Private Key는 master에 보관하고,
Public Key를 접속 대상 Node에 등록한다.

## 3. Public Key 전달
```bash
ssh-copy-id ubuntu@10.86.202.84
ssh-copy-id ubuntu@10.86.202.137
```
각 worker의 ~/.ssh/authorized_keys에
master의 Public Key가 등록된다.

## 4. 비밀번호 없이 SSH 접속 확인
```bash
ssh ubuntu@10.86.202.84
ssh ubuntu@10.86.202.137
```

비밀번호를 추가로 입력하지 않고 접속할 수 있으면
SSH Key 기반 인증이 정상적으로 구성된 것이다.

## 5. Hostname 설정

각 Node의 이름을 확인한다.

hostname

예:

master
worker1
worker2
## 6. /etc/hosts 설정

각 Node에서:
```bash
cat /etc/hosts
```
Node 이름과 IP를 등록한다.

예:

10.86.202.17   master
10.86.202.84   worker1
10.86.202.137  worker2

이후:
```bash
ping master
ping worker1
ping worker2
```
와 같이 hostname을 이용한 통신이 가능하다.

## 7. SSH 연결 확인

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

## 8. Ansible에서 SSH 확인

Kubespray 디렉터리에서:
```bash
ansible-inventory \
  -i inventory/mycluster/hosts.yaml \
  --graph
```
Inventory가 정상적으로 구성되었는지 확인한다.

그 다음:
```bash
ansible all \
  -i inventory/mycluster/hosts.yaml \
  -m ping
```
각 Node에서:

pong

이 반환되면 Ansible을 통한 SSH 연결이 정상적으로 동작한다.

## 9. Kubespray와 SSH의 관계

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
