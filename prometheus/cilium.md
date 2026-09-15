이제 cilium과 연결해서 보자.

kubectl get pods -n kube-system -l k8s-app=cilium -o wide

NAME           READY   STATUS    RESTARTS        AGE    IP              NODE      NOMINATED NODE   READINESS GATES
cilium-cw8t5   1/1     Running   10 (107m ago)   4d2h   10.86.202.137   worker2   <none>           <none>
cilium-r28jw   1/1     Running   10 (107m ago)   4d2h   10.86.202.84    worker1   <none>           <none>
cilium-v6vsd   1/1     Running   6 (107m ago)    4d2h   10.86.202.17    master    <none>           <none>

Cilium Operator의 Prometheus metrics endpoint가 어디에 열려있나 확인

kubectl get configmap -n kube-system cilium-config -o yaml | grep -i -A3 -B3 prometheus

  node-port-bind-protection: "true"
  nodes-gc-interval: 5m0s
  operator-api-serve-addr: 127.0.0.1:9234
  operator-prometheus-serve-addr: :9963
  packetization-layer-pmtud-mode: blackhole
  policy-default-local-cluster: "true"
  policy-deny-response: none

보면 Metrics가 켜져 있지 않음.

kubectl -n kube-system exec cilium-cw8t5 -- cilium-dbg status

Hubble:  Ok  Current/Max Flows: 4095/4095 (100.00%), Flows/s: 6.96   Metrics: Disabled
보면 Metrics가 disabled되어있음.

kubectl get svc -n kube-system | grep hubble

이걸로 hubble관한 서비스를 잡아내봤을 때 hubble-peer이 나온다. 이 hubble-peer은 

worker1 Cilium ──┐
                 ├── hubble-peer ── Cilium 간 flow 전달
worker2 Cilium ──┤
master  Cilium ──┘

Helm values가 최소 설정으로 들어가 있어 metric 기능을 켜서 해보자

https://docs.cilium.io/en/stable/observability/metrics/#hubble-metrics
여기 들어가보면 cilium의 metrics기능을 어떻게 키는지 알려준다.

helm upgrade cilium cilium/cilium \
  --version 1.20.1 \
  -n kube-system \
  --reuse-values \
  --set prometheus.enabled=true \
  --set hubble.enabled=true \
  --set hubble.metrics.enableOpenMetrics=true \
  --set 'hubble.metrics.enabled={dns,drop,tcp,flow,port-distribution,icmp}'
  
이렇게 이미 설치되어있으니 upgrade하는 쪽, reuse-values하는 쪽으로 hubble.metrics.enableOpenMetrics=true
로 바꾼 것이 핵심.

helm get values cilium -n kube-system
다시 해보면

USER-SUPPLIED VALUES:
cluster:
  name: cluster-local
hubble:
  enabled: true
  metrics:
    enableOpenMetrics: true
    enabled:
    - dns
    - drop
    - tcp
    - flow
    - port-distribution
    - icmp
operator:
  replicas: 1
prometheus:
  enabled: true
routingMode: tunnel
tunnelProtocol: vxlan

metircs의 설정이 드러난 것을 볼 수 있다.

cilium자체가 재부팅되면서 pod의 이름도 달라졌다.
적용해서 cilium의 metrics의 포트를 찾아보자

kubectl exec -n kube-system cilium-5hphb -- ss -lnt

LISTEN 0      4096               *:9962             *:*

실제로 사용하는 9962포트가 열린 것을 확인할 수 있다.


