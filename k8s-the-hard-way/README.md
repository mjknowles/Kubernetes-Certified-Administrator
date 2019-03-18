# Kubernetes the Hard Way Summary

## Install prereq tools

* cfssl:  generates encryption keys

```bash
wget -q --show-progress --https-only --timestamping \
  https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
  https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
```

## Networking

1. Provision a virtual private cloud network with a 'custom' subnet mode because we will be providing our own subnets.

2. Create a subnet inside the VPC network large enough for addresses for each *node* in the k8s cluster. Ex. 10.240.0.0/24 which can hold up to 254 nodes.

3. Create firewall rule to allow internal commuication of all protocols across the network and certain external protocols.

4. Reserve a public internet address for your load balancer

## Nodes

1. Create master node compute instances. Also referred to as "controllers". It will belong to the subnet created earlier.
   
2. Create worker node compute instances. Same subnet as master nodes.

## Public Key Infrastructure

1. Provision the Certificate Authority

```
{

cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}
```
This generates ca-key.pem and ca.pem

2. Generate a key/certificate pair for an admin user
3. Generate a key/certificate pair for each worker kubelet
4. Genreate a key/certificate pair for the for the Controller Manager client
5. Generate the Kube Proxy certificate/key pair
6. Generate the Kube Scheduler certificate/key pair
7. Genreate the Kubernetes API Server certificate/key pair
8. Generate the certificate/key pair for the Controller Manager's Service Account generation
9. Copy worker node kubelet cert/key pairs to the root directory of the worker nodes
10. Copy CA key/cert, API key/cert, and service account generation key/cert to the root directory of the master nodes

## Kubeconfig Generation

Kubeconfigs allow clients to locate and authenticate with the API server.

1. Generate kubeconfig files for worker nodes

```
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${hostname}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
```

2. Generate the kube-proxy kubeconfig
3. Generate the kube-controller-manager kubeconfig
4. Generate the kube-scheduler kubeconfig
5. Generate a kubeconfig for the admin user
6. Copy kubelet and kube-proxy kubeconfig to worker nodes to the root directory of the nodes
7. Copy the kube-controller-manager and kube-scheduler kubeconfigs to the master nodes to the root directory of the nodes

## Data Encryption

1. Generate the encryption key

```
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```
2. Create encryption-config.yaml file
```
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```
3. Copy encryption config file to root directory of master nodes

## Bootstrap etcd

1. SSH into each master node in the cluster and do this
2. Download and extract the etcd binaries and etcdctl command line utility
3. Copy ca.pem, K8s API key, and K8s API .pem to /etc/etcd
4. Create etcd.service systemd unit file:
```
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://10.240.0.10:2380,controller-1=https://10.240.0.11:2380,controller-2=https://10.240.0.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
5. Start the etcd server
```
{
  sudo systemctl daemon-reload
  sudo systemctl enable etcd
  sudo systemctl start etcd
}
```

## Control Plane
1. SSH into each master node in the cluster and do this
2. Download and install the API, controller-manager, scheduler, and kubectl binaries
3. Copy the CA.pem, CA-key.pem, service-account.pem, and encryption-config.yaml files to /var/lib/kubernetes
4. Create the kube-apiserver.service systemd unit file
```
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 \\
  --event-ttl=1h \\
  --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=api/all \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
5. Move kube-controller-manager.kubeconfig to /var/liv/kubernetes
6. Create the kube-controller-manager.service file in /etc/systemd/system
8. Move kube-scheduler.kubeconfig to /var/lib/kubernetes
9.  Create the kube-scheduler.yaml config file at /etc/kubernetes/config
10. Create the kube-scheduler.service file in /etc/systemd/system
11. Start the controller services
```
{
  sudo systemctl daemon-reload
  sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
}
```
12. Check the status of components
```
kubectl get componentstatuses --kubeconfig admin.kubeconfig
```

## Authorize Kubelet

1. Create a ClusterRole with permissions to access the Kubelet API
2. Create and apply a ClusterRoleBinding for the role created above and the 'kubernetes' account

## Bootstrapping a Worker Node

1. Install socat, conntrack, and ipset