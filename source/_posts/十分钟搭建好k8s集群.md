---
title: 十分钟搭建好K8S集群
tags:
  - dashboard
  - k8s
  - kubeadm
  - kubernates
id: '1236'
categories:
  - - 微服务
date: 2020-12-22 14:14:32
---

虽然网上有大量从零搭建K8S的文章，但大都针对老版本，若直接照搬去安装最新的1.20版本会遇到一堆问题。故此将我的安装步骤记录下来，希望能为读者提供copy and paste式的集群搭建帮助。 我是在腾讯云CentOS的2台服务器上，在不翻墙的情况下使用kubeadm（最简单的部署工具）搭建K8S集群。其中，容器网络使用flannel，最终安装好可以外网访问的2.0.4版本的dashboard。不走弯路的话，4步即可完成：

<!-- more -->

## 准备系统环境

对这2台新购的云虚拟机，都需要安装docker：

```
yum install docker -y
```

安装kubeadm前，需要先添加kubeadm的yum源：

```
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
EOF
```

接着安装目前最新版的kubeadm：

```
yum install kubeadm-1.20.1-0.x86_64
```

其中，kubeadm依赖的kubelet（K8S最核心的组件）、kubectl、kubernetes-cni网络插件等会自动安装。 最后启用这2个服务：

```
systemctl enable kubelet.service
systemctl start docker.service
```

腾讯云主机默认已经将bridge-nf-call-iptables和ip\_forward置为1，如果你的主机值为0，需要先将功能开启：

```
sysctl -w net.bridge.bridge-nf-call-iptables=1
sysctl -w net.ipv4.ip_forward=1
```

接着，针对部署K8S master节点的主机，还需要做些工作。由于腾讯云默认的主机名并不是域名格式（不符合master节点要求），所以需要修改为你自定义的域名。这需要2个步骤：

1.  将设计好的主机名写入/etc/hostname文件；
2.  将/etc/hosts中原主机名替换掉。

 

## 部署master节点

kubeadm的init命令即可初始化以单节点部署的master。为了避免翻墙，我**使用了`registry.aliyuncs.com/google_containers`源**。虽然可以在kubeadm命令行中输入源，比如：

```
kubeadm init --image-repository registry.aliyuncs.com/google_containers
```

但将其写入资源编排文件更易维护，我将其命名为kubeadm.yaml：

```
apiVersion: kubeadm.k8s.io/v1beta2
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.20.1
networking:
  podSubnet: 10.244.0.0/16
controllerManager:
        ExtraArgs:
                horizontal-pod-autoscaler-use-rest-clients: "true"
                horizontal-pod-autoscaler-sync-period: "10s"
                node-monitor-grace-period: "10s"
```

注意，这个编排文件有4个细节：

*   接口版本使用了v1beta2；
*   为flannel分配的网段是`10.244.0.0/16`；
*   选择的kubernetes版本是当前最新的1.20.1；
*   加入了`controllerManager`的水平扩容功能。

接着，使用编排文件执行init命令：

```
kubeadm init --config ./kubeadm.yaml
```

注意，执行成功后kubeadm会返回类似下面的字样：

```
Your Kubernetes control-plane has initialized successfully!
```

之后，还有2条重要的信息： 1、后续kubectl默认控制集群时，需要使用到CA密钥，通过TLS协议保障通讯的安全性。我们要通过下面3行命令拷贝密钥信息，这样，kubectl执行时会首先访问当前用户的.kube目录，使用这些授权信息访问集群：

```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

2、之后添加worker节点时，要通过token才能保障安全性。因此，先把显示的这行命令保存下来：

```
kubeadm join 172.27.0.11:6443 --token xxxxxxxxxxxx \
    --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxx
```

 

## 部署flannel网络

flannel网络需要指定IP地址段，在上一步中已经通过编排文件设置为`10.244.0.0/16`。部署flannel只需要一行命令：

```
kubectl apply -f ./kube-flannel.yml
```

其中，由于原本可以直接下载的文件必须翻墙，所以我把文件内容列在下方，保存为上述命令的kube-flannel.yml文件（参见第5小节）： 此时，调用get pods命令，可以看到master节点的组件和flannel网络都牌Running状态：

```
[root@k8s]# kubectl get pods -n kube-system
NAME                                  READY   STATUS    RESTARTS   AGE
coredns-7f89b7bc75-g8c42              1/1     Running   0          118s
coredns-7f89b7bc75-z2p7c              1/1     Running   0          118s
etcd-taohui.tech                      1/1     Running   0          2m6s
kube-apiserver-taohui.tech            1/1     Running   0          2m6s
kube-controller-manager-taohui.tech   1/1     Running   0          2m6s
kube-flannel-ds-kfzrg                 1/1     Running   0          13s
kube-proxy-qbdz5                      1/1     Running   0          118s
kube-scheduler-taohui.tech            1/1     Running   0          2m6s
```

接着，登入另一台服务器上，通过之前保存的kubeadm join命令，将其加入K8S集群。通过get nodes命令可以看到集群中已有2个节点：

```
# kubectl get nodes
NAME              STATUS   ROLES                  AGE   VERSION
taohui.tech       Ready    control-plane,master   14h   v1.20.0
workernode        Ready    <none>                 12h   v1.20.1
```

至此集群部署成功！如果有参数错误需要修改，你也可以在reset后重新init集群：

```
kubeadm reset
```

 

## 部署dashboard的

我们可以用以WEB页面的可视化dashboard，监控集群的状态。部署时同样面临翻墙和版本匹配的问题，这里我将2个镜像源替换后保存为dashboard.yaml编排文件（参见第6小节）：

```
kubectl apply -f dashboard.yaml
```

执行成功后，你可以查看到service：

```
# kubectl get svc -n kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
dashboard-metrics-scraper   ClusterIP   10.100.6.83     <none>        8000/TCP   13h
kubernetes-dashboard        ClusterIP   10.99.243.226   <none>        443/TCP    13h
```

注意，dashboard默认不允许外网访问。即使我们通过kubectl proxy允许外网访问，但dashboard又只允许HTTPS访问，这样kubeadm init时自签名的CA证书是不被浏览器承认的。我采用的方案是Nginx作为反向代理，使用Lets Encrypt提供的有效证书对外提供服务，再经由proxy\_pass指令反向代理到kubectl proxy上，例如：

```
kubectl proxy --port=8888 --accept-hosts='^*$'
```

此时，本地可经由8888访问到dashboard。再通过Nginx访问它：

```
proxy_pass http://localhost:8888;
```

这样，当在外网根据我的域名访问如下URL：

```
https://mydomain/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login
```

就会展现2种登陆方式： [![](/2020/12/k8s_dashboard_login.jpg)](http://www.taohui.pub/2020/12/22/%e5%8d%81%e5%88%86%e9%92%9f%e6%90%ad%e5%bb%ba%e5%a5%bdk8s%e9%9b%86%e7%be%a4/k8s_dashboard_login/) 接下来采用token方式。首先要创建管理员帐户：

```
cat <<EOF  kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

执行完后，serviceaccount/admin-user用户已经创建。接着，将用户绑定已经存在的集群管理员角色：

```
cat <<EOF  kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

最后，获取可用户于访问的token：

```
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret  grep admin-user  awk '{print $1}')
```

将token粘贴到登录页面，就可以看到dashboard了（下图是node页面）： [![](/2020/12/k8s_dashboard_node.jpg)](http://www.taohui.pub/2020/12/22/%e5%8d%81%e5%88%86%e9%92%9f%e6%90%ad%e5%bb%ba%e5%a5%bdk8s%e9%9b%86%e7%be%a4/k8s_dashboard_node/)

## 用于部署flannel网络的资源编排文件

```
---
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp.flannel.unprivileged
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: docker/default
    seccomp.security.alpha.kubernetes.io/defaultProfileName: docker/default
    apparmor.security.beta.kubernetes.io/allowedProfileNames: runtime/default
    apparmor.security.beta.kubernetes.io/defaultProfileName: runtime/default
spec:
  privileged: false
  volumes:
  - configMap
  - secret
  - emptyDir
  - hostPath
  allowedHostPaths:
  - pathPrefix: "/etc/cni/net.d"
  - pathPrefix: "/etc/kube-flannel"
  - pathPrefix: "/run/flannel"
  readOnlyRootFilesystem: false
  # Users and groups
  runAsUser:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  # Privilege Escalation
  allowPrivilegeEscalation: false
  defaultAllowPrivilegeEscalation: false
  # Capabilities
  allowedCapabilities: ['NET_ADMIN', 'NET_RAW']
  defaultAddCapabilities: []
  requiredDropCapabilities: []
  # Host namespaces
  hostPID: false
  hostIPC: false
  hostNetwork: true
  hostPorts:
  - min: 0
    max: 65535
  # SELinux
  seLinux:
    # SELinux is unused in CaaSP
    rule: 'RunAsAny'
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
rules:
- apiGroups: ['extensions']
  resources: ['podsecuritypolicies']
  verbs: ['use']
  resourceNames: ['psp.flannel.unprivileged']
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: 
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: 
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      hostNetwork: true
      priorityClassName: system-node-critical
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: quay.io/coreos/flannel:v0.13.1-rc1
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.13.1-rc1
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
```

 

## 用于部署dashboard的资源编排文件dashboard.yaml

```
# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

apiVersion: v1
kind: Namespace
metadata:
  name: kubernetes-dashboard

---

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard

---

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kubernetes-dashboard
type: Opaque

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-csrf
  namespace: kubernetes-dashboard
type: Opaque
data:
  csrf: ""

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-key-holder
  namespace: kubernetes-dashboard
type: Opaque

---

kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-settings
  namespace: kubernetes-dashboard

---

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
rules:
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs", "kubernetes-dashboard-csrf"]
    verbs: ["get", "update", "delete"]
    # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["kubernetes-dashboard-settings"]
    verbs: ["get", "update"]
    # Allow Dashboard to get metrics.
  - apiGroups: [""]
    resources: ["services"]
    resourceNames: ["heapster", "dashboard-metrics-scraper"]
    verbs: ["proxy"]
  - apiGroups: [""]
    resources: ["services/proxy"]
    resourceNames: ["heapster", "http:heapster:", "https:heapster:", "dashboard-metrics-scraper", "http:dashboard-metrics-scraper"]
    verbs: ["get"]

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
rules:
  # Allow Metrics Scraper to get metrics from the Metrics server
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list", "watch"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard

---

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
        - name: kubernetes-dashboard
          image: registry.cn-shanghai.aliyuncs.com/jieee/dashboard:v2.0.4
          imagePullPolicy: Always
          ports:
            - containerPort: 8443
              protocol: TCP
          args:
            - --auto-generate-certificates
            - --namespace=kubernetes-dashboard
            # Uncomment the following line to manually specify Kubernetes API server Host
            # If not specified, Dashboard will attempt to auto discover the API server and connect
            # to it. Uncomment only if the default does not work.
            # - --apiserver-host=http://my-address:port
          volumeMounts:
            - name: kubernetes-dashboard-certs
              mountPath: /certs
              # Create on-disk volume to store exec logs
            - mountPath: /tmp
              name: tmp-volume
          livenessProbe:
            httpGet:
              scheme: HTTPS
              path: /
              port: 8443
            initialDelaySeconds: 30
            timeoutSeconds: 30
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      volumes:
        - name: kubernetes-dashboard-certs
          secret:
            secretName: kubernetes-dashboard-certs
        - name: tmp-volume
          emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule

---

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 8000
      targetPort: 8000
  selector:
    k8s-app: dashboard-metrics-scraper

---

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: dashboard-metrics-scraper
  template:
    metadata:
      labels:
        k8s-app: dashboard-metrics-scraper
      annotations:
        seccomp.security.alpha.kubernetes.io/pod: 'runtime/default'
    spec:
      containers:
        - name: dashboard-metrics-scraper
          image: registry.cn-shanghai.aliyuncs.com/jieee/metrics-scraper:v1.0.4
          ports:
            - containerPort: 8000
              protocol: TCP
          livenessProbe:
            httpGet:
              scheme: HTTP
              path: /
              port: 8000
            initialDelaySeconds: 30
            timeoutSeconds: 30
          volumeMounts:
          - mountPath: /tmp
            name: tmp-volume
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      volumes:
        - name: tmp-volume
          emptyDir: {}
```