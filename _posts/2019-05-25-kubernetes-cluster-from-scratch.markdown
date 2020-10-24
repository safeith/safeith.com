---
layout: post
title:  "Kubernetes cluster from scratch"
date:   2019-05-25 10:19:07 +0430
categories: Kubernetes
---
__<span style="color: red;">Note: This document is deprecated</span>__

In this document we will create a kubernetes cluster with Ubuntu 18.04 servers and Hyperkube, It's a multi master cluster.

## Requirements
1. Two Ubuntu 18.04 as master
2. Two Ubuntu 18.4 as worker
3. Minimum 2 core vCPU per server
4. Minimum 2 GB RAM per server

## PreConfiguration
Setup DNS resolver for servers in own client

{% highlight bash %}
nano /etc/hosts
# Add your servers like following lines
10.10.0.2 k8s-controller-1
10.10.0.3 k8s-controller-2
10.10.0.4 k8s-worker-1
10.10.0.5 k8s-worker-1
{% endhighlight %}

Setup date/time in all servers

{% highlight bash %}
for i in k8s-controller-1 k8s-controller-2 k8s-worker-1 k8s-worker-2
do
  ssh root@$i "timedatectl set-local-rtc 0; timedatectl set-timezone UTC"
done
{% endhighlight %}

Setup DNS resolver for servers in all servers

{% highlight bash %}
for i in k8s-controller-1 k8s-controller-2 k8s-worker-1 k8s-worker-2
do
  grep -w k8s /etc/hosts | ssh root@$i "tee -a /etc/hosts"
done
{% endhighlight %}

Setup bridge netfilter and IP forwarding

{% highlight bash %}
for i in k8s-controller-1 k8s-controller-2 k8s-worker-1 k8s-worker-2
do
  echo -e "net.bridge.bridge-nf-call-iptables=1\n\
  net.bridge.bridge-nf-call-ip6tables=1\n\
  net.ipv4.ip_forward=1" \
  | ssh root@$i "tee /etc/sysctl.d/kubernetes.conf && \
  modprobe br_netfilter && sysctl -p --system"
done
{% endhighlight %}

## Create cerficates

Login to first master  with ssh

{% highlight bash %}
ssh root@k8s-controller-1 -p 22
{% endhighlight %}

Create openssl configuration

{% highlight bash %}
CONTROLLER1_IP=$(getent ahostsv4 k8s-controller-1 | tail -1 | awk '{print $1}')
CONTROLLER2_IP=$(getent ahostsv4 k8s-controller-2 | tail -1 | awk '{print $1}')
SERVICE_IP="10.96.0.1"

mkdir -p /etc/kubernetes/pki

cd /etc/kubernetes/pki

cat > openssl.cnf << EOF
[ req ]
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_ca ]
basicConstraints = critical, CA:TRUE
keyUsage = critical, digitalSignature, keyEncipherment, keyCertSign
[ v3_req_server ]
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
[ v3_req_client ]
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
[ v3_req_apiserver ]
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names_cluster
[ v3_req_etcd ]
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names_etcd
[ alt_names_cluster ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
DNS.5 = k8s-controller-1
DNS.6 = k8s-controller-2
# DNS.7 = ${KUBERNETES_PUBLIC_ADDRESS}
IP.1 = ${CONTROLLER1_IP}
IP.2 = ${CONTROLLER2_IP}
IP.3 = ${SERVICE_IP}
# IP.4 = ${KUBERNETES_PUBLIC_IP}
[ alt_names_etcd ]
DNS.1 = k8s-controller-1
DNS.2 = k8s-controller-2
IP.1 = ${CONTROLLER1_IP}
IP.2 = ${CONTROLLER2_IP}
EOF
{% endhighlight %}

Create kubernetes CA certificate

{% highlight bash %}
openssl ecparam -name secp521r1 -genkey -noout -out ca.key

chmod 0600 ca.key

openssl req -x509 -new -sha256 -nodes -key ca.key -days 3650 -out ca.crt \
-subj "/CN=kubernetes-ca"  -extensions v3_ca \
-config ./openssl.cnf
{% endhighlight %}

Create kube apiserver certificate

{% highlight bash %}
openssl ecparam -name secp521r1 -genkey -noout -out kube-apiserver.key

chmod 0600 kube-apiserver.key

openssl req -new -sha256 -key kube-apiserver.key -subj "/CN=kube-apiserver" \
| openssl x509 -req -sha256 -CA ca.crt -CAkey ca.key -CAcreateserial \
-out kube-apiserver.crt -days 365 \
-extensions v3_req_apiserver \
-extfile ./openssl.cnf
{% endhighlight %}

Create apiserver kubelet client certificate

{% highlight bash %}
openssl ecparam -name secp521r1 -genkey -noout -out apiserver-kubelet-client.key

chmod 0600 apiserver-kubelet-client.key

openssl req -new -key apiserver-kubelet-client.key \
-subj "/CN=kube-apiserver-kubelet-client/O=system:masters" \
| openssl x509 -req -sha256 -CA ca.crt -CAkey ca.key -CAcreateserial \
-out apiserver-kubelet-client.crt -days 365 \
-extensions v3_req_client \
-extfile ./openssl.cnf
{% endhighlight %}

Create admin client certificate

{% highlight bash %}
openssl ecparam -name secp521r1 -genkey -noout -out admin.key

chmod 0600 admin.key

openssl req -new -key admin.key -subj "/CN=kubernetes-admin/O=system:masters" \
| openssl x509 -req -sha256 -CA ca.crt -CAkey ca.key -CAcreateserial \
-out admin.crt -days 365 -extensions v3_req_client \
-extfile ./openssl.cnf
{% endhighlight %}

Create service account key

{% highlight bash %}
openssl ecparam -name secp521r1 -genkey -noout -out sa.key

openssl ec -in sa.key -outform PEM -pubout -out sa.pub

chmod 0600 sa.key

openssl req -new -sha256 -key sa.key \
-subj "/CN=system:kube-controller-manager" \
| openssl x509 -req -sha256 -CA ca.crt -CAkey ca.key -CAcreateserial \
-out sa.crt -days 365 -extensions v3_req_client \
-extfile ./openssl.cnf
{% endhighlight %}

Create kube-scheduler certificate

{% highlight bash %}
openssl ecparam -name secp521r1 -genkey -noout -out kube-scheduler.key

chmod 0600 kube-scheduler.key

openssl req -new -sha256 -key kube-scheduler.key \
-subj "/CN=system:kube-scheduler" \
| openssl x509 -req -sha256 -CA ca.crt -CAkey ca.key -CAcreateserial \
-out kube-scheduler.crt -days 365 -extensions v3_req_client \
-extfile ./openssl.cnf
{% endhighlight %}

Create front proxy CA certificate

{% highlight bash %}
openssl ecparam -name secp521r1 -genkey -noout -out front-proxy-ca.key

chmod 0600 front-proxy-ca.key

openssl req -x509 -new -sha256 -nodes -key front-proxy-ca.key -days 3650 \
-out front-proxy-ca.crt -subj "/CN=front-proxy-ca" \
-extensions v3_ca \
-config ./openssl.cnf
{% endhighlight %}

Create front proxy client certificate

{% highlight bash %}
openssl ecparam -name secp521r1 -genkey -noout -out front-proxy-client.key

chmod 0600 front-proxy-client.key

openssl req -new -sha256 -key front-proxy-client.key \
-subj "/CN=front-proxy-client" \
| openssl x509 -req -sha256 -CA front-proxy-ca.crt \
-CAkey front-proxy-ca.key -CAcreateserial \
-out front-proxy-client.crt -days 365 \
-extensions v3_req_client \
-extfile ./openssl.cnf
{% endhighlight %}

Create kube-proxy certificate

{% highlight bash %}
openssl ecparam -name secp521r1 -genkey -noout -out kube-proxy.key

chmod 0600 kube-proxy.key

openssl req -new -key kube-proxy.key \
-subj "/CN=kube-proxy/O=system:node-proxier" \
| openssl x509 -req -sha256 -CA ca.crt -CAkey ca.key -CAcreateserial \
-out kube-proxy.crt -days 365 -extensions v3_req_client \
-extfile ./openssl.cnf
{% endhighlight %}

Create etcd CA certificate

{% highlight bash %}
openssl ecparam -name secp521r1 -genkey -noout -out etcd-ca.key

chmod 0600 etcd-ca.key

openssl req -x509 -new -sha256 -nodes -key etcd-ca.key -days 3650 \
-out etcd-ca.crt -subj "/CN=etcd-ca" -extensions v3_ca \
-config ./openssl.cnf
{% endhighlight %}

Create etcd certificate

{% highlight bash %}
openssl ecparam -name secp521r1 -genkey -noout -out etcd.key

chmod 0600 etcd.key

openssl req -new -sha256 -key etcd.key -subj "/CN=etcd" \
| openssl x509 -req -sha256 -CA etcd-ca.crt -CAkey etcd-ca.key \
-CAcreateserial -out etcd.crt -days 365 \
-extensions v3_req_etcd \
-extfile ./openssl.cnf
{% endhighlight %}

Create etcd peer cert

{% highlight bash %}
openssl ecparam -name secp521r1 -genkey -noout -out etcd-peer.key

chmod 0600 etcd-peer.key

openssl req -new -sha256 -key etcd-peer.key -subj "/CN=etcd-peer" \
| openssl x509 -req -sha256 -CA etcd-ca.crt -CAkey etcd-ca.key \
-CAcreateserial -out etcd-peer.crt -days 365 \
-extensions v3_req_etcd \
-extfile ./openssl.cnf
{% endhighlight %}

View created certificates

{% highlight bash %}
for i in *crt
do
  echo $i:
  openssl x509 -subject -issuer -noout -in $i
  echo
done
{% endhighlight %}

Copy certificates to another controller

{% highlight bash %}
ssh -o StrictHostKeyChecking=no root@k8s-controller-2 "mkdir /etc/kubernetes"
scp -pr -- /etc/kubernetes/pki/ k8s-controller-2:/etc/kubernetes/
cd ~
{% endhighlight %}

## Install binaries
Install these binaries in all controllers

{% highlight bash %}
TAG=v1.10.2
URL=https://storage.googleapis.com/kubernetes-release/release/$TAG/bin/linux/amd64
curl -# -L -o /usr/bin/hyperkube $URL/hyperkube
chmod +x /usr/bin/hyperkube
cd /usr/bin/
hyperkube --make-symlinks
chmod +x kube*
cd ~
{% endhighlight %}

## Generate kubernetes configs
### Generate kubeconfig files on all controller nodes

Create service account kubeconfig

{% highlight bash %}
CONTROLLER1_IP=$(getent ahostsv4 k8s-controller-1 | tail -1 | awk '{print $1}')
INTERNAL_IP=$(hostname -I | awk '{print $1}')
KUBERNETES_PUBLIC_ADDRESS=$INTERNAL_IP
CLUSTER_NAME="default"
KCONFIG=controller-manager.kubeconfig
KUSER="system:kube-controller-manager"
KCERT=sa

cd /etc/kubernetes/

kubectl config set-cluster ${CLUSTER_NAME} \
--certificate-authority=pki/ca.crt \
--embed-certs=true \
--server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
--kubeconfig=${KCONFIG}

kubectl config set-credentials ${KUSER} \
--client-certificate=pki/${KCERT}.crt \
--client-key=pki/${KCERT}.key \
--embed-certs=true \
--kubeconfig=${KCONFIG}

kubectl config set-context ${KUSER}@${CLUSTER_NAME} \
--cluster=${CLUSTER_NAME} \
--user=${KUSER} \
--kubeconfig=${KCONFIG}

kubectl config use-context ${KUSER}@${CLUSTER_NAME} --kubeconfig=${KCONFIG}

kubectl config view --kubeconfig=${KCONFIG}
{% endhighlight %}

Create kube-scheduler kubeconfig

{% highlight bash %}
CONTROLLER1_IP=$(getent ahostsv4 k8s-controller-1 | tail -1 | awk '{print $1}')
INTERNAL_IP=$(hostname -I | awk '{print $1}')
KUBERNETES_PUBLIC_ADDRESS=$INTERNAL_IP
CLUSTER_NAME="default"
KCONFIG=scheduler.kubeconfig
KUSER="system:kube-scheduler"
KCERT=kube-scheduler

cd /etc/kubernetes/

kubectl config set-cluster ${CLUSTER_NAME} \
--certificate-authority=pki/ca.crt \
--embed-certs=true \
--server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
--kubeconfig=${KCONFIG}

kubectl config set-credentials ${KUSER} \
--client-certificate=pki/${KCERT}.crt \
--client-key=pki/${KCERT}.key \
--embed-certs=true \
--kubeconfig=${KCONFIG}

kubectl config set-context ${KUSER}@${CLUSTER_NAME} \
--cluster=${CLUSTER_NAME} \
--user=${KUSER} \
--kubeconfig=${KCONFIG}

kubectl config use-context ${KUSER}@${CLUSTER_NAME} --kubeconfig=${KCONFIG}

kubectl config view --kubeconfig=${KCONFIG}
{% endhighlight %}

Create admin kubeconfig

{% highlight bash %}
CONTROLLER1_IP=$(getent ahostsv4 k8s-controller-1 | tail -1 | awk '{print $1}')
INTERNAL_IP=$(hostname -I | awk '{print $1}')
KUBERNETES_PUBLIC_ADDRESS=$INTERNAL_IP
CLUSTER_NAME="default"
KCONFIG=admin.kubeconfig
KUSER="kubernetes-admin"
KCERT=admin

cd /etc/kubernetes/

kubectl config set-cluster ${CLUSTER_NAME} \
--certificate-authority=pki/ca.crt \
--embed-certs=true \
--server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
--kubeconfig=${KCONFIG}

kubectl config set-credentials ${KUSER} \
--client-certificate=pki/${KCERT}.crt \
--client-key=pki/${KCERT}.key \
--embed-certs=true \
--kubeconfig=${KCONFIG}

kubectl config set-context ${KUSER}@${CLUSTER_NAME} \
--cluster=${CLUSTER_NAME} \
--user=${KUSER} \
--kubeconfig=${KCONFIG}

kubectl config use-context ${KUSER}@${CLUSTER_NAME} --kubeconfig=${KCONFIG}

kubectl config view --kubeconfig=${KCONFIG}
{% endhighlight %}

## Deploy etcd
> Do this in first conroller

Install etcd binaries

{% highlight bash %}
cd ~
TAG=v3.3.4
URL=https://github.com/coreos/etcd/releases/download/$TAG
curl -# -LO $URL/etcd-$TAG-linux-amd64.tar.gz
tar xvf etcd-$TAG-linux-amd64.tar.gz
chown -Rh root:root etcd-$TAG-linux-amd64/
find etcd-$TAG-linux-amd64/ -xdev -type f -exec chmod 0755 '{}' \;
cp etcd-$TAG-linux-amd64/etcd* /usr/bin/
mkdir -p /var/lib/etcd
{% endhighlight %}

Create systemd for etcd

{% highlight bash %}
CONTROLLER1_IP=$(getent ahostsv4 k8s-controller-1 | tail -1 | awk '{print $1}')
INTERNAL_IP=$(hostname -I | awk '{print $1}')
ETCD_CLUSTER_TOKEN="default-27a5f27fe2" # this should be unique per cluster
ETCD_NAME='k8s-controller-1'
ETCD_CERT_FILE=/etc/kubernetes/pki/etcd.crt
ETCD_CERT_KEY_FILE=/etc/kubernetes/pki/etcd.key
ETCD_PEER_CERT_FILE=/etc/kubernetes/pki/etcd-peer.crt
ETCD_PEER_KEY_FILE=/etc/kubernetes/pki/etcd-peer.key
ETCD_CA_FILE=/etc/kubernetes/pki/etcd-ca.crt
ETCD_PEER_CA_FILE=/etc/kubernetes/pki/etcd-ca.crt

cat > /etc/systemd/system/etcd.service << EOF
[Unit]
Description=etcd
Documentation=https://coreos.com/etcd/docs/latest/
After=network.target
Befor=kube-apiserver.service

[Service]
ExecStart=/usr/bin/etcd \\
  --name ${ETCD_NAME} \\
  --listen-client-urls https://${INTERNAL_IP}:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --data-dir=/var/lib/etcd \\
  --cert-file=${ETCD_CERT_FILE} \\
  --key-file=${ETCD_CERT_KEY_FILE} \\
  --peer-cert-file=${ETCD_PEER_CERT_FILE} \\
  --peer-key-file=${ETCD_PEER_KEY_FILE} \\
  --trusted-ca-file=${ETCD_CA_FILE} \\
  --peer-trusted-ca-file=${ETCD_CA_FILE} \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --initial-cluster-token ${ETCD_CLUSTER_TOKEN} \\
  --initial-cluster k8s-controller-1=https://${CONTROLLER1_IP}:2380 \\
  --initial-cluster-state new
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
systemctl status etcd -l
{% endhighlight %}

Verify etcd is working
With etcdctl command
{% highlight bash %}
etcdctl \
  --ca-file=/etc/kubernetes/pki/etcd-ca.crt \
  --cert-file=/etc/kubernetes/pki/etcd.crt \
  --key-file=/etc/kubernetes/pki/etcd.key \
  cluster-health

etcdctl \
  --ca-file=/etc/kubernetes/pki/etcd-ca.crt \
  --cert-file=/etc/kubernetes/pki/etcd.crt \
  --key-file=/etc/kubernetes/pki/etcd.key \
  member list
{% endhighlight %}

Verify etcd is working with openssl

{% highlight bash %}
echo -e "GET /health HTTP/1.1\nHost: $INTERNAL_IP\n" \
  | timeout 2s openssl s_client -CAfile /etc/kubernetes/pki/etcd-ca.crt \
  -cert /etc/kubernetes/pki/etcd.crt \
  -key /etc/kubernetes/pki/etcd.key \
  -connect $INTERNAL_IP:2379 \
  -ign_eof
{% endhighlight %}

## Create systemd for Kubernetes API Server
Do this in all controllers

{% highlight bash %}
CONTROLLER1_IP=$(getent ahostsv4 k8s-controller-1 | tail -1 | awk '{print $1}')
INTERNAL_IP=$(hostname -I | awk '{print $1}')
SERVICE_CLUSTER_IP_RANGE="10.96.0.0/12"

cat > /etc/systemd/system/kube-apiserver.service << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target etcd.service

[Service]
ExecStart=/usr/bin/hyperkube apiserver \\
  --apiserver-count=2 \\
  --allow-privileged=true \\
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,ResourceQuota,DefaultTolerationSeconds \\
  --authorization-mode=RBAC \\
  --secure-port=6443 \\
  --bind-address=0.0.0.0 \\
  --advertise-address=${INTERNAL_IP} \\
  --insecure-port=0 \\
  --insecure-bind-address=127.0.0.1 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/kube-audit.log \\
  --client-ca-file=/etc/kubernetes/pki/ca.crt \\
  --etcd-cafile=/etc/kubernetes/pki/etcd-ca.crt \\
  --etcd-certfile=/etc/kubernetes/pki/etcd.crt \\
  --etcd-keyfile=/etc/kubernetes/pki/etcd.key \\
  --etcd-servers=https://${CONTROLLER1_IP}:2379 \\
  --service-account-key-file=/etc/kubernetes/pki/sa.pub \\
  --service-cluster-ip-range=${SERVICE_CLUSTER_IP_RANGE} \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/etc/kubernetes/pki/kube-apiserver.crt \\
  --tls-private-key-file=/etc/kubernetes/pki/kube-apiserver.key \\
  --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt \\
  --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key \\
  --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname \\
  --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt \\
  --requestheader-username-headers=X-Remote-User \\
  --requestheader-group-headers=X-Remote-Group \\
  --requestheader-allowed-names=front-proxy-client \\
  --requestheader-extra-headers-prefix=X-Remote-Extra- \\
  --v=2
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver -l
{% endhighlight %}

## Create systemd for Kubernetes Controller Manager
Do this in all controllers

{% highlight bash %}
CLUSTER_CIDR="192.168.0.0/16"
SERVICE_CLUSTER_IP_RANGE="10.96.0.0/12"
CLUSTER_NAME="default"

cat > /etc/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
After=network.target kube-apiserver.service

[Service]
ExecStart=/usr/bin/hyperkube controller-manager \\
  --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig \\
  --address=127.0.0.1 \\
  --leader-elect=true \\
  --controllers=*,bootstrapsigner,tokencleaner \\
  --service-account-private-key-file=/etc/kubernetes/pki/sa.key \\
  --insecure-experimental-approve-all-kubelet-csrs-for-group=system:bootstrappers \\
  --cluster-cidr=${CLUSTER_CIDR} \\
  --allocate-node-cidrs=true \\
  --cluster-name=${CLUSTER_NAME} \\
  --service-cluster-ip-range=${SERVICE_CLUSTER_IP_RANGE} \\
  --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt \\
  --cluster-signing-key-file=/etc/kubernetes/pki/ca.key \\
  --root-ca-file=/etc/kubernetes/pki/ca.crt \\
  --use-service-account-credentials=true \\
  --v=2
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
systemctl status kube-controller-manager -l
{% endhighlight %}

## Create systemd for Kubernetes Scheduler
Do this in all controllers

{% highlight bash %}
cat > /etc/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
After=network.target kube-controller-manager.service

[Service]
ExecStart=/usr/bin/hyperkube scheduler \\
  --leader-elect=true \\
  --kubeconfig=/etc/kubernetes/scheduler.kubeconfig \\
  --address=127.0.0.1 \\
  --v=2
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
systemctl status kube-scheduler -l
{% endhighlight %}

## Verify the controllers
Do this in all controllers

{% highlight bash %}
export KUBECONFIG=/etc/kubernetes/admin.kubeconfig
kubectl version
kubectl get componentstatuses
{% endhighlight %}

## Create kubectl bash completion
Do this in all controllers

{% highlight bash %}
echo 'export KUBECONFIG=/etc/kubernetes/admin.kubeconfig' >> .bashrc
echo 'source <(kubectl completion bash)' >> .bashrc
source .bashrc
{% endhighlight %}

## Generate bootstrap token
Do this in one of controllers and save TOKEN_PUB, TOKEN_SECRET and  BOOTSTRAP_TOKEN in secured places

{% highlight bash %}
TOKEN_PUB=$(openssl rand -hex 3)
echo $TOKEN_PUB

TOKEN_SECRET=$(openssl rand -hex 8)
echo $TOKEN_SECRET

BOOTSTRAP_TOKEN="${TOKEN_PUB}.${TOKEN_SECRET}"
echo $BOOTSTRAP_TOKEN

kubectl -n kube-system create secret generic bootstrap-token-${TOKEN_PUB} \
--type 'bootstrap.kubernetes.io/token' \
--from-literal description="cluster bootstrap token" \
--from-literal token-id=${TOKEN_PUB} \
--from-literal token-secret=${TOKEN_SECRET} \
--from-literal usage-bootstrap-authentication=true \
--from-literal usage-bootstrap-signing=true

kubectl -n kube-system get secret/bootstrap-token-${TOKEN_PUB} -o yaml
{% endhighlight %}

## Create bootstrap kubeconfig
Do this in one of controllers

{% highlight bash %}
INTERNAL_IP=$(hostname -I | awk '{print $1}')
KUBERNETES_PUBLIC_ADDRESS=$INTERNAL_IP

CLUSTER_NAME="default"
KCONFIG="bootstrap.kubeconfig"
KUSER="kubelet-bootstrap"

cd /etc/kubernetes

kubectl config set-cluster ${CLUSTER_NAME} \
--certificate-authority=pki/ca.crt \
--embed-certs=true \
--server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
--kubeconfig=${KCONFIG}

kubectl config set-context ${KUSER}@${CLUSTER_NAME} \
--cluster=${CLUSTER_NAME} \
--user=${KUSER} \
--kubeconfig=${KCONFIG}

kubectl config use-context ${KUSER}@${CLUSTER_NAME} --kubeconfig=${KCONFIG}

kubectl config view --kubeconfig=${KCONFIG}
{% endhighlight %}

## Expose CA and bootstrap kubeconfig via configmap
Do this in one of controllers
> Make sure the bootstrap kubeconfig file does not contain the bootstrap token
before you expose it via the cluster-info configmap.

{% highlight bash %}
kubectl -n kube-public create configmap cluster-info \
--from-file /etc/kubernetes/pki/ca.crt \
--from-file /etc/kubernetes/bootstrap.kubeconfig
{% endhighlight %}

Allow anonymous user to acceess the cluster-info configmap.
Do this in one of controllers

{% highlight bash %}
kubectl -n kube-public create role system:bootstrap-signer-clusterinfo \
--verb get --resource configmaps

kubectl -n kube-public create rolebinding kubeadm:bootstrap-signer-clusterinfo \
--role system:bootstrap-signer-clusterinfo --user system:anonymous
{% endhighlight %}

Allow a bootstrapping worker node join the cluster.
Do this in one of controllers

{% highlight bash %}
kubectl create clusterrolebinding kubeadm:kubelet-bootstrap \
--clusterrole system:node-bootstrapper --group system:bootstrappers
{% endhighlight %}

## Install Docker
Do this in all of servers

{% highlight bash %}
apt install docker.io
{% endhighlight %}

Edit docker systemd file and check ExecStart to

{% highlight bash %}
/usr/bin/dockerd --iptables=false --storage-driver overlay -H fd://

{% endhighlight %}
Then restart docker's systemd

{% highlight bash %}
systemctl daemon-reload
systemctl restart docker.Service
{% endhighlight %}

## Install Kubernetes binaries in workers
Do this in all of workers

{% highlight bash %}
TAG=v1.10.2
URL=https://storage.googleapis.com/kubernetes-release/release/$TAG/bin/linux/amd64
curl -# -L -o /usr/bin/hyperkube $URL/hyperkube
chmod +x /usr/bin/hyperkube
cd /usr/bin/
hyperkube --make-symlinks
chmod +x kube*
cd ~
{% endhighlight %}

## Retrieve CA and the bootstrap kubeconfig
Do this in all workers

{% highlight bash %}
mkdir -p /etc/kubernetes/pki

kubectl -n kube-public get cm/cluster-info \
--server https://k8s-controller-1:6443 --insecure-skip-tls-verify=true \
--username=system:anonymous --output=jsonpath='{.data.ca\.crt}' \
| tee /etc/kubernetes/pki/ca.crt

kubectl -n kube-public get cm/cluster-info \
--server https://k8s-controller-1:6443 --insecure-skip-tls-verify=true \
--username=system:anonymous \
--output=jsonpath='{.data.bootstrap\.kubeconfig}' \
| tee /etc/kubernetes/bootstrap.kubeconfig
{% endhighlight %}

Now write previously generated BOOTSTRAP_TOKEN to the bootstrap kubeconfig

{% highlight bash %}
read -r -s -p "BOOTSTRAP_TOKEN: " BOOTSTRAP_TOKEN

kubectl config set-credentials kubelet-bootstrap \
--token=${BOOTSTRAP_TOKEN} \
--kubeconfig=/etc/kubernetes/bootstrap.kubeconfig
{% endhighlight %}

## Install CNI plugins
Do this in all servers
> Need to find latest version

{% highlight bash %}
mkdir -p /etc/cni/net.d /opt/cni

ARCH=amd64
CNI_RELEASE=0799f5732f2a11b329d9e3d51b9c8f2e3759f2ff
URL=https://storage.googleapis.com/kubernetes-release/network-plugins
curl -sSL $URL/cni-${ARCH}-${CNI_RELEASE}.tar.gz | tar -xz -C /opt/cni
{% endhighlight %}

## Create systemd for Kubernetes Kubelet
Do this all of server
> For master nodes do

{% highlight bash %}
cp -rv /etc/kubernetes/admin.kubeconfig /etc/kubernetes/kubelet.conf
{% endhighlight %}

> For worker nodes do

{% highlight bash %}
for i in k8s-worker-1 k8s-worker-2; do
  scp -p -- /etc/kubernetes/admin.kubeconfig $i:/etc/kubernetes/kubelet.conf
done
{% endhighlight %}

{% highlight bash %}
CLUSTER_DNS_IP=10.96.0.10

mkdir -p /etc/kubernetes/manifests

cat > /etc/systemd/system/kubelet.service << EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service kube-scheduler.service
Requires=docker.service

[Service]
ExecStart=/usr/bin/hyperkube kubelet \\
  --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \\
  --kubeconfig=/etc/kubernetes/kubelet.conf \\
  --pod-manifest-path=/etc/kubernetes/manifests \\
  --allow-privileged=true \\
  --network-plugin=cni \\
  --cni-conf-dir=/etc/cni/net.d \\
  --cni-bin-dir=/opt/cni/bin \\
  --cluster-dns=${CLUSTER_DNS_IP} \\
  --cluster-domain=cluster.local \\
  --authorization-mode=Webhook \\
  --client-ca-file=/etc/kubernetes/pki/ca.crt \\
  --cgroup-driver=cgroupfs \\
  --cert-dir=/etc/kubernetes
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet -l
{% endhighlight %}

> Make controller nodes unschedulable by any pods

{% highlight bash %}
for i in k8s-controller-1 k8s-controller-2
do
kubectl label node $i node-role.kubernetes.io/master=
kubectl taint nodes $i node-role.kubernetes.io/master=:NoSchedule
done
{% endhighlight %}

## Install kube-proxy
Create a kube-proxy service account in one of controllers

{% highlight bash %}
kubectl -n kube-system create serviceaccount kube-proxy
{% endhighlight %}

Create a kube-proxy kubeconfig

{% highlight bash %}
INTERNAL_IP=$(hostname -I | awk '{print $1}')
KUBERNETES_PUBLIC_ADDRESS=$INTERNAL_IP

export KUBECONFIG=/etc/kubernetes/admin.kubeconfig

SECRET=$(kubectl -n kube-system get sa/kube-proxy \
 --output=jsonpath='{.secrets[0].name}')

JWT_TOKEN=$(kubectl -n kube-system get secret/$SECRET \
--output=jsonpath='{.data.token}' | base64 -d)

CLUSTER_NAME="default"
KCONFIG="kube-proxy.kubeconfig"

cd /etc/kubernetes

kubectl config set-cluster ${CLUSTER_NAME} \
--certificate-authority=pki/ca.crt \
--embed-certs=true \
--server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
--kubeconfig=${KCONFIG}

kubectl config set-context ${CLUSTER_NAME} \
--cluster=${CLUSTER_NAME} \
--user=default \
--namespace=default \
--kubeconfig=${KCONFIG}

kubectl config set-credentials ${CLUSTER_NAME} \
--token=${JWT_TOKEN} \
--kubeconfig=${KCONFIG}

kubectl config use-context ${CLUSTER_NAME} --kubeconfig=${KCONFIG}

kubectl config view --kubeconfig=${KCONFIG}
{% endhighlight %}

Bind a kube-proxy service account (from kube-system namespace) to a clusterrole system:node-proxier to allow RBAC

{% highlight bash %}
kubectl create clusterrolebinding kubeadm:node-proxier \
--clusterrole system:node-proxier \
--serviceaccount kube-system:kube-proxy
{% endhighlight %}

Copy kube-proxy.kubeconfig to workers

{% highlight bash %}
for i in k8s-worker-1 k8s-worker-2
do
scp -p -- /etc/kubernetes/kube-proxy.kubeconfig $i:/etc/kubernetes/
done
{% endhighlight %}

Create systemd for kube-proxy

{% highlight bash %}
mkdir /var/lib/kube-proxy

cat > /etc/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes
After=network.target kubelet.service

[Service]
ExecStart=/usr/bin/hyperkube proxy \\
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \\
  --v=2
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy -l
{% endhighlight %}

## Deploy Calico
Do this in one of controllers

{% highlight bash %}
kubectl apply -f \
https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml

kubectl apply -f \
https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
{% endhighlight %}

## Deploy kube-DNS
Do This in one of controllers

{% highlight bash %}
cd /etc/kubernetes/manifests/

wget https://raw.githubusercontent.com/kelseyhightower/kubernetes-the-hard-way/master/deployments/kube-dns.yaml

sed -i 's/clusterIP: 10.32.0.10/clusterIP: 10.96.0.10/g' kube-dns.yaml

kubectl create -f kube-dns.yaml
{% endhighlight %}

> YOUR WELCOME  :-)
