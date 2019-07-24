# docker install

```
sudo yum install -y yum-utils device-mapper-persistent-data lvm2

sudo yum-config-manager \
     --add-repo \
     https://download.docker.com/linux/centos/docker-ce.repo

sudo yum makecache fast

sudo yum install docker-ce-18.03.1.ce-1.el7.centos -y


vim /usr/lib/systemd/system/docker.service 加上--graph "/alidata1/dockerdata"，更改存储路径

ExecStart=/usr/bin/dockerd --graph "/alidata1/dockerdata"

sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker

```

更改镜像

```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://xxx.mirror.aliyuncs.com"]
}
EOF

```

自己去阿里云申请镜像地址

# k8s images国内下载

```
docker pull mirrorgooglecontainers/kube-apiserver:v1.15.0
docker pull mirrorgooglecontainers/kube-controller-manager:v1.15.0
docker pull mirrorgooglecontainers/kube-scheduler:v1.15.0
docker pull mirrorgooglecontainers/kube-proxy:v1.15.0
docker pull mirrorgooglecontainers/pause:3.1
docker pull mirrorgooglecontainers/etcd:3.3.10
docker pull coredns/coredns:1.3.1

docker tag mirrorgooglecontainers/kube-apiserver:v1.15.0 k8s.gcr.io/kube-apiserver:v1.15.0
docker tag mirrorgooglecontainers/kube-controller-manager:v1.15.0 k8s.gcr.io/kube-controller-manager:v1.15.0
docker tag mirrorgooglecontainers/kube-scheduler:v1.15.0 k8s.gcr.io/kube-scheduler:v1.15.0
docker tag mirrorgooglecontainers/kube-proxy:v1.15.0 k8s.gcr.io/kube-proxy:v1.15.0
docker tag mirrorgooglecontainers/pause:3.1 k8s.gcr.io/pause:3.1
docker tag mirrorgooglecontainers/etcd:3.3.10 k8s.gcr.io/etcd:3.3.10
docker tag coredns/coredns:1.3.1 k8s.gcr.io/coredns:1.3.1

```

```
kubeadm init --pod-network-cidr=192.168.0.0/16
kubectl apply -f https://docs.projectcalico.org/v3.7/manifests/calico.yaml
```

一定要注意k8s的版本和docker的版本，要不然后面kubeadm init时你会崩溃的

# 独立coredns配置

```
---
apiVersion: v1
kind: Service
metadata:
  name: xxx-dns
  namespace: cdns
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: xxx-dns
spec:
  clusterIP: 10.96.107.114
  selector:
    k8s-app: xxx-dns
  ports:
  - name: dns
    nodePort: 31888
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    nodePort: 31888
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP
  type: NodePort
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: xxxdns
  namespace: cdns
  labels:
    k8s-app: xxx-dns
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: xxx-dns
  template:
    metadata:
      labels:
        k8s-app: xxx-dns
    spec:
      serviceAccountName: xxxdns
      nodeSelector:
        beta.kubernetes.io/os: linux
      containers:
      - name: xxxdns
        image: coredns/coredns:1.5.0
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: xxxdns
            items:
            - key: Corefile
              path: Corefile
apiVersion: v1
kind: Namespace
metadata:
   name: cdns
   labels:
     name: cdns
apiVersion: v1
kind: ServiceAccount
metadata:
  name: xxxdns
  namespace: cdns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:xxxdns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:xxxdns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:xxxdns
subjects:
- kind: ServiceAccount
  name: xxxdns
  namespace: cdns
apiVersion: v1
kind: ConfigMap
metadata:
  name: xxxdns
  namespace: cdns
data:
  Corefile: |
    .:53 {
      errors
      health
      hosts {
        10.10.10.10 xx.xx.com
        10.10.10.11 xx.xx.com
        10.10.10.12 xx.xx.com
        fallthrough
      }
      kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        upstream
        fallthrough in-addr.arpa ip6.arpa
      }
      prometheus :9153
      forward . 114.114.114.114 
      cache 30
      reload
    }

```

nginx配置：

```
stream {
        upstream udp_proxy {
        server 127.0.0.1:31888;
   }
      server {
        listen 53 udp;
        proxy_pass udp_proxy;
     }
  }
```










