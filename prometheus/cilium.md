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

kubectl get svc -n kube-system | grep -Ei 'cilium|hubble'

cilium-envoy                                         ClusterIP   None            <none>        9964/TCP                       4d23h
hubble-metrics                                       ClusterIP   None            <none>        9965/TCP                       19h
hubble-peer                                          ClusterIP   10.233.53.176   <none>        443/TCP                        4d23h

실제로 사용하는 9965포트가 열린 것을 확인할 수 있다.

이제 prometheus가 어느 때 serviceMonitor을 선택하는지 알아보자.

kubectl -n monitoring get prometheus -o yaml | grep -A10 -B5 serviceMonitorSelector

runAsUser: 1000
      seccompProfile:
        type: RuntimeDefault
    serviceAccountName: monitoring-kube-prometheus-prometheus
    serviceMonitorNamespaceSelector: {}
    serviceMonitorSelector:
      matchLabels:
        release: monitoring
    shards: 1
    tsdb:
      outOfOrderTimeWindow: 0s
    version: v3.14.0-distroless
    walCompression: true
  status:
    availableReplicas: 1
    conditions:

이렇게 ServiceMonitorSlelctor은 label에 monitoring이 붙어 있어야 scrape 대상으로 삼는다.

그럼 huble-metrics에는 어느 label이 붙는 지 확인해 보면
kubectl -n kube-system get svc hubble-metrics -o yaml

...
  labels:
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: hubble
    app.kubernetes.io/part-of: cilium
    helm.sh/chart: cilium-1.20.1
    k8s-app: hubble

이렇게 hubble임을 알 수 있음. 따라서 label에 hubble이 붙어 있으면 scrape대상으로 삼을 수 있도록 바꿔준다.

nano hubble-servicemonitor.yaml

만들어서

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: hubble-metrics
  namespace: kube-system
  labels:
    release: monitoring
spec:
  selector:
    matchLabels:
      k8s-app: hubble
  endpoints:
    - port: hubble-metrics
      path: /metrics
      interval: 15s

ServieceMonitor이라는 k8s객체를 만들고 여기에 release : monitoring이라는 label을 붙여서 
Prometheus가 관리할 수 있도록 한다.
그럼 이 객체는 selector가 Label이 k8s-app: hubble인 놈들을 선택한다는 설정.

kubectl apply -f hubble-servicemonitor.yaml
kubectl get servicemonitor -n kube-system

히먄
NAME             AGE
hubble-metrics   7s

servicemonitor이라는 Kubernetes 리소스가 hubble-metrics라는 이름으로 생성된다.

Prometheus에

{__name__=~"hubble_.*"}

입력하면 이제 관찰이 가능하다.
