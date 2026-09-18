kubectl port-forward \
  --address 0.0.0.0 \
  -n monitoring \
  svc/monitoring-grafana \
  3000:80


ssh \
  -i ~/.ssh/multipass_id_rsa \
  -N \
  -L 0.0.0.0:3000:127.0.0.1:3000 \
  ubuntu@10.86.202.17

두개 해서 windows에 연결하고
kubectl get secret -n monitoring monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d
echo

해서
eXp2ti29FyNxgmqG1qr59Z0LOJpuMDD5HFHgbRSh
