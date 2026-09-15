ubuntu master에서 helm을 설치해주고
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
prometheus community repository추가
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
이거 설치만 하면 원래는
Prometheus
Grafana
Alertmanager
node-exporter
kube-state-metrics
요거까지 다 가능
개별적으로 다 설치하지 말고 처음에는 kube-prometheus-stack을 쓴다

Monitoring Namespace만들기
```bash
kubectl create namespace monitoring
kubectl get ns
```
kube-prometheus-stack 설치
```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring
```
확인
```bash
kubectl get pod -n monitoring -o wide
```
그러면
```text
NAME                                                     READY   STATUS    RESTARTS        AGE     IP              NODE      NOMINATED NODE   READINESS GATES
alertmanager-monitoring-kube-prometheus-alertmanager-0   2/2     Running   2 (5m7s ago)    73m     10.0.2.199      worker1   <none>           <none>
monitoring-grafana-684cfc98b7-g67zt                      3/3     Running   3 (5m7s ago)    60m     10.0.2.62       worker1   <none>           <none>
monitoring-kube-prometheus-operator-594bc8666-p7slz      1/1     Running   1 (5m7s ago)    73m     10.0.2.188      worker1   <none>           <none>
monitoring-kube-state-metrics-764584797b-n795n           1/1     Running   2 (5m7s ago)    73m     10.0.2.17       worker1   <none>           <none>
monitoring-prometheus-node-exporter-47psd                1/1     Running   1 (10m ago)     73m     10.86.202.137   worker2   <none>           <none>
monitoring-prometheus-node-exporter-rqnj7                1/1     Running   1 (8m15s ago)   73m     10.86.202.17    master    <none>           <none>
monitoring-prometheus-node-exporter-xr26r                1/1     Running   1 (5m7s ago)    73m     10.86.202.84    worker1   <none>           <none>
prometheus-monitoring-kube-prometheus-prometheus-0       2/2     Running   0               7m24s 10.0.2.32       worker1   <none>           <none>
```
이렇게 나온다.
node-exporter는 DaemonSet이기 때문에 각 Kubernetes node마다 하나씩 뜬다.

Prometheus가 실제로 데이터를 받고 있는지 확인
```bash
kubectl get svc -n monitoring
```
```text
NAME                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
alertmanager-operated                     ClusterIP   None            <none>        9093/TCP,9094/TCP,9094/UDP   78m
monitoring-grafana                        ClusterIP   10.233.15.64    <none>        80/TCP                       78m
monitoring-kube-prometheus-alertmanager   ClusterIP   10.233.34.250   <none>        9093/TCP,8080/TCP            78m
monitoring-kube-prometheus-operator       ClusterIP   10.233.31.3     <none>        443/TCP                      78m
monitoring-kube-prometheus-prometheus     ClusterIP   10.233.61.241   <none>        9090/TCP,8080/TCP            78m
monitoring-kube-state-metrics             ClusterIP   10.233.7.196    <none>        8080/TCP                     78m
monitoring-prometheus-node-exporter       ClusterIP   10.233.46.213   <none>        9100/TCP                     78m
prometheus-operated                       ClusterIP   None            <none>        9090/TCP                     78m
```
Prometheus를 외부에서 접근하기 쉽게 일단 port-forward 하자.
```bash
kubectl port-forward \
  --address 0.0.0.0 \
  -n monitoring \
  svc/monitoring-kube-prometheus-prometheus \
  9090:9090
```
이렇게 master의 어느 ip로 접근하든 9090포트로 접근하면 svc쪽 prometheus의 9090포트와 연결된다.
그런데 지금 환경은 WSL2이므로 windows에서 WSL2안의 VM인 master node까지 접근하기 위해서는
ssh 터널링이 필요하다.
```bash
ssh \
  -i ~/.ssh/multipass_id_rsa \
  -N \
  -L 0.0.0.0:9090:127.0.0.1:9090 \
  ubuntu@10.86.202.17
```
이렇게 WSL로 향하는 ssh접근이 곧 9090포트로 이러지면서 ubuntu의 master node의 ip로 접근이 가능

그러면 WSL상에서
```bash
curl http://172.21.195.79:9090/-/ready
```
하면
```text
Prometheus Server is Ready.
```
이렇게 뜨는 것을 볼 수 있다. 요 ip는 Windows에서 보는 WSL의 주요 ip이다.

windows powershell에서
```powershell
curl http://172.21.195.79:9090/-/ready
```
실행하면
```text
StatusCode        : 200
StatusDescription : OK
Content           : Prometheus Server is Ready.

RawContent        : HTTP/1.1 200 OK
                    Content-Length: 28
                    Content-Type: text/plain; charset=utf-8
                    Date: Tue, 15 Sep 2026 06:07:35 GMT

                    Prometheus Server is Ready.

Forms             : {}
Headers           : {[Content-Length, 28], [Content-Type, text/plain; charse
                    t=utf-8], [Date, Tue, 15 Sep 2026 06:07:35 GMT]}
Images            : {}
InputFields       : {}
Links             : {}
ParsedHtml        : mshtml.HTMLDocumentClass
RawContentLength  : 28
```

이렇게 뜨는 것을 확인할 수 있다.
```text
Prometheus 정상 접근 완료!
```
그러면
http://172.21.195.79:9090/
요 웹으로 접근할 수 있다.

prometheus에
```bash
100 - (
  avg by (instance) (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  ) * 100
)
```
넣고 execute하면
cpu사용량을 볼 수 있음.
```text
{instance="10.86.202.137:9100"}	9.357407407407422
{instance="10.86.202.17:9100"}	13.694444444444457
{instance="10.86.202.84:9100"}	7.609259259259261
```
```bash
100 * (
  1 - (
    node_memory_MemAvailable_bytes
    /
    node_memory_MemTotal_bytes
  )
)
```
넣고 execute하면
메모리 사용량을 볼 수 있음.
```text
{container="node-exporter", endpoint="http-metrics", instance="10.86.202.137:9100", job="node-exporter", namespace="monitoring", pod="monitoring-prometheus-node-exporter-47psd", service="monitoring-prometheus-node-exporter"}	31.512767229810134
{container="node-exporter", endpoint="http-metrics", instance="10.86.202.17:9100", job="node-exporter", namespace="monitoring", pod="monitoring-prometheus-node-exporter-rqnj7", service="monitoring-prometheus-node-exporter"}	34.34504952079234
{container="node-exporter", endpoint="http-metrics", instance="10.86.202.84:9100", job="node-exporter", namespace="monitoring", pod="monitoring-prometheus-node-exporter-xr26r", service="monitoring-prometheus-node-exporter"}  25.516159290867236
```
