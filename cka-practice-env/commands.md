kubectl run exam-nginx --image=nginx:1.12.2 --replicas=2 --record
kubectl get deploy exam-nginx -o yaml --export > nginx-deploy.yml
kubectl replace -f nginx-deploy.yml --save-config

kubectl get componentstatuses

sudo systemctl daemon-reload
sudo systemctl enable containerd kubelet kube-proxy
sudo systemctl start containerd kubelet kube-proxy

curl -i https://3.202.26.8:6443/apis --cacert ca.pem --key service-account-key.pem --cert service-account.pem

journalctl -u kube-scheduler

sudo ETCDCTL_API=3 etcdctl snapshot save snapshotdb --endpoints=https://127.0.0.1:2379   --cacert=/etc/etcd/ca.pem   --cert=/etc/etcd/kubernetes.pem   --key=/etc/etcd/kubernetes-key.pem

sudo -i
