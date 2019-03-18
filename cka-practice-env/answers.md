1. 
master:
```
kubeadm init --kubernetes-version 1.10
kubeadm token list
```
worker:
```
kubeadm join --discovery-token-unsafe-skip-ca-verification --token=<token> <master_node_ip>:6443
```
2. I don't know.
3. Create deployment based on example from k8s deployment page. or

kubectl run exam-nginx --image=nginx:1.12.2 --replicas=2 --record
kubectl get deploy exam-nginx -o yaml --export > nginx-deploy.yml
kubectl replace -f nginx-deploy.yml --save-config

Manually add an annotations:
    kubernetes.io/change-cause: "image updated to 1.13.8"

kubectl apply -f <config>.yml

kubectl rollout status deployment exam-nginx

kubectl rollout undo deployment exam-nginx --to-revision=3

4. First create volume ("gce-volume.yml") then PVC then add reference in deploy script

kubectl exec -it task-pv-pod -- /bin/bash
