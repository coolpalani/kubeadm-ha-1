kubeadm ha

// install keepalived
yum install -y keepalived; systemctl enable keepalived && systemctl restart keepalived
vi /etc/keepalived/check_apiserver.sh
#!/bin/bash
err=0
for k in $( seq 1 10 )
do
    check_code=$(ps -ef|grep kube-apiserver |grep -v grep | wc -l)
    if [ "$check_code" = "1" ]; then
        err=$(expr $err + 1)
        sleep 5
        continue
    else
        err=0
        break
    fi
done
if [ "$err" != "0" ]; then
    echo "systemctl stop keepalived"
    /usr/bin/systemctl stop keepalived
    exit 1
else
    exit 0
fi

chmod a+x /etc/keepalived/check_apiserver.sh

vi /etc/keepalived/keepalive.conf

! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 2
    weight -5
    fall 3  
    rise 2
}
vrrp_instance VI_1 {
    state MASTER
    interface ens160
    mcast_src_ip 192.168.50.211
    virtual_router_id 55
    priority 100
    advert_int 2
    virtual_ipaddress {
        192.168.50.209
    }
    track_script {
       chk_apiserver
    }
}

// nginx load balancer
user  nginx;
worker_processes auto;

error_log  /var/log/nginx/error.log info;
pid        /var/run/nginx.pid;

load_module /usr/lib64/nginx/modules/ngx_stream_module.so;

events {
    worker_connections  1024;
}

stream {

    upstream apiserver {
        server 192.168.50.210:6443 weight=5 max_fails=3 fail_timeout=30s;
        server 192.168.0.211:6443 weight=5 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 8443;
        proxy_pass apiserver;
    }
}


// install_etcd.sh
#!/bin/bash
declare -a etcd_hosts_ip=("192.168.50.210" "192.168.50.211" "192.168.50.212")
declare -a etcd_hosts
function join_by { local IFS="$1"; shift; echo "$*"; }

echo ${#etcd_hosts_ip[@]}
for (( i=0; i<${#etcd_hosts_ip[@]};i++));
do
    etcd_hosts[$i]=etcd$i\=http://${etcd_hosts_ip[$i]}:2380
done
etcd_cluster=$(join_by , "${etcd_hosts[@]}")
echo "etcd cluster: "$etcd_cluster

let i=0
for h in "${etcd_hosts_ip[@]}";
do
    node=$h
    name=etcd$i
    let i=i+1
    ssh root@$node /bin/bash << EOF 
    docker stop etcd; docker rm etcd
    rm -fr /var/lib/etcd
    rm -rf /var/lib/etcd-cluster
    mkdir -p /var/lib/etcd-cluster
    docker run -d \
    --restart always \
    -v /etc/ssl/certs:/etc/ssl/certs \
    -v /var/lib/etcd-cluster:/var/lib/etcd \
    -p 4001:4001 \
    -p 2380:2380 \
    -p 2379:2379 \
    --name etcd \
    gcr.io/google_containers/etcd-amd64:3.1.11 \
    etcd --name=$name \
    --advertise-client-urls=http://$node:2379,http://$node:4001 \
    --listen-client-urls=http://0.0.0.0:2379,http://0.0.0.0:4001 \
    --initial-advertise-peer-urls=http://$node:2380 \
    --listen-peer-urls=http://0.0.0.0:2380 \
    --initial-cluster=$etcd_cluster \
    --initial-cluster-token=etc-cluster-0 \
    --initial-cluster-state=new \
    --auto-tls \
    --peer-auto-tls \
    --data-dir=/var/lib/etcd
EOF
done


// verify
docker exec -ti etcd ash
etcdctl member list
etcdctl cluster-health


// kubeadm init configuration file kubeadm.yaml
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
kubernetesVersion: v1.9.0
networking:
  podSubnet: 10.244.0.0/16
api:
  advertiseAddress: 192.168.50.210
apiServerCertSANs:
- node1
- node0
- 192.168.50.210
- 192.168.50.211
- 192.168.50.212
- 192.168.50.209
selfHosted: true
#featureGates:
#  CoreDNS: true
tokenTtl: 99999h
etcd:
  endpoints:
  - http://192.168.50.210:2379
  - http://192.168.50.211:2379
  - http://192.168.50.212:2379
controllerManagerExtraArgs:
  address: 0.0.0.0
schedulerExtraArgs:
  address: 0.0.0.0

kubeadm init --config kubeadm.yaml

kubeadm join --token a6887e.3c986342091bf7b9 192.168.50.209:8443 --discovery-token-ca-cert-hash sha256:f3ea9db7ea41b2be9651bfe45657b9fa5fd0330df6af4fb4bf917d128a91b664

// install kube-router
KUBECONFIG=/etc/kubernetes/admin.conf kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter-all-features.yaml
KUBECONFIG=/etc/kubernetes/admin.conf kubectl -n kube-system delete ds kube-proxy;
docker run --privileged --net=host gcr.io/google_containers/kube-proxy-amd64:v1.7.3 kube-proxy --cleanup-iptables

// or install flannel
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

// modify admission-control
vi /etc/kubernetes/manifests/kube-apiserver.yaml
replayce with:
- --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,DefaultTolerationSeconds

systemctl restart docker kubelet

// add other master nodes cp master /etc/kubernetes to other masters
scp -r /etc/kubernetes root@node0:/etc/

// relpace all hostname and ip with new one

// restart 
systemctl daemon-reload; systemctl restart docker kubelet

// scale kubedns to num of masters

// change apiserver addr tp vip address
// pache kube-proxy
kubectl get -n kube-system configmap/kube-proxy \
  -o yaml >/tmp/kube-proxy.yaml

kubectl edit configmap/kube-proxy -n kube-system
// server: https://${master_vip}:${lb_port}
server: https://192.168.50.209:8443


// patch master as unschedulable
// kubectl patch node node1 -p '{"spec":{"unschedulable":true}}'
// kubectl patch node node0 -p '{"spec":{"unschedulable":true}}'

// add worker nodes (via secondary master node)
kubeadm token list
kubeadm join $(vip):8443 --token bafca3.cf535e5dcc5ffff4 --discovery-token-unsafe-skip-ca-verification
vi /etc/kubernetes/bootstrap-kubelet.conf change server to server: https://$(vip):8443

// or 
sed -ie 's|server: .*|server: https://192.168.50.209:8443|' /etc/kubernetes/kubelet.conf && systemctl restart kubelet

// add new worker node via secondary master node:
kubeadm join --token a6f9e8.e754d000893baa9c 192.168.50.209:8443 --discovery-token-unsafe-skip-ca-verification

// apply kubedashboard
 kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml

// apply nginx-ingress-controller
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/namespace.yaml \
    | kubectl apply -f -

curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/default-backend.yaml \
    | kubectl apply -f -

curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/configmap.yaml \
    | kubectl apply -f -

curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/tcp-services-configmap.yaml \
    | kubectl apply -f -

curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/udp-services-configmap.yaml \
    | kubectl apply -f -

curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/rbac.yaml \
    | kubectl apply -f -

curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/with-rbac.yaml \
    | kubectl apply -f -

// patch kube-dashboard with affinity spec, constrain installation on edge routers
// kubectl patch deploy nginx-ingress-controller -n ingress-nginx  -p "$(cat patch-deployment.yaml.9)"
spec:
  template:    
    spec:
      hostNetwork: true
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - node5
  

// apply kube-lego


// patch node authrization
kubectl create clusterrolebinding node0 --clusterrole=cluster-admin --user=system:node:node0
kubectl create clusterrolebinding node1 --clusterrole=cluster-admin --user=system:node:node1


// modifications for prometheus-operator
change address from 127.0.0.1 to 0.0.0.0 in etc/kubernetes/manifests/kube-scheduler.yaml, and kube-controller-manager.yaml
//and expose cAdvisor on all nodes
sed -e "/cadvisor-port=0/d" -i /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
systemctl daemon-reload
systemctl restart kubelet
