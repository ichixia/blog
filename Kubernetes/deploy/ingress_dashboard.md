### k8s使用ingress让集群外部通过域名访问kube-dashboard    
k8s的service虽然可以把部署的pods提供给k8s集群外部访问， 但是通过把service设置成NodePort类型，直接使用宿主机的端口的形式不太好，一个是端口比较有限，另外一个是不好管理  

这里我们通过ingress的方式，让k8s集群外部通过指定的域名访问我们的kube-dashboard，主要的步骤有下面几步：  

1. 部署Nginx Ingress Controller
2. 创建与dashboard service关联的ingress资源
3. 第一步部署的Nginx Ingress Controller就会自动发现这个ingress资源从而去重载自己的配置  

#### 环境与准备  
1. 单机k8s环境，核心组件都准备好了
2. k8s 插件准备(flannel、coredns、kubernetes-dashboard)
3. kubernetes-dashboard工作正常，集群可以正常访问dashboard  

```
[root@vm-node1 ingress]# kubectl get pods -n kube-system
NAME                               READY   STATUS    RESTARTS   AGE
coredns-6955765f44-4c2ht           1/1     Running   0          3d13h
coredns-6955765f44-t5nbk           1/1     Running   0          3d13h
etcd-vm-node1                      1/1     Running   0          3d13h
kube-apiserver-vm-node1            1/1     Running   0          3d13h
kube-controller-manager-vm-node1   1/1     Running   0          3d13h
kube-flannel-ds-amd64-rhghn        1/1     Running   0          3d13h
kube-proxy-x9229                   1/1     Running   0          3d13h
kube-scheduler-vm-node1            1/1     Running   0          3d13h

[root@vm-node1 ingress]# kubectl get pods -n kubernetes-dashboard
NAME                                         READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-7b64584c5c-wwphv   1/1     Running   0          3d
kubernetes-dashboard-b7ffbc8cb-4bsbs         1/1     Running   0          3d

[root@vm-node1 ingress]# kubectl get svc -n kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.96.8.151   <none>        8000/TCP        3d
kubernetes-dashboard        NodePort    10.96.17.2    <none>        443:32443/TCP   3d
```

#### 开始正活 
这里部署Ingress Controller的方式是DaemonSet+HostNetwork+nodeSelector  
##### 1. 通过pods部署nginx-ingress-controller  
执行一个简单的yaml,官方的yaml文件地址是`https://github.com/kubernetes/ingress-nginx/tree/master/deploy/static/mandatory.yaml`  

这里部署的方式是DaemonSet+HostNetwork+nodeSelector，所以修改了了一下yaml文件  

在执行yaml文件之前，先给我的单机k8s的node打上标签，这样才能部署导致这个node上面  
```
kubectl label node vm-node1 isIngress="true"
```

然后就可以部署这个ingress-controller了  
```
kubectl apply -f mandatory.yaml  
```  

下面是我的yaml文件的内容  
<details>
    <summary>点击这里查看yaml文件内容</summary>  
    
    apiVersion: v1
    kind: Namespace
    metadata:
      name: ingress-nginx
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
    ---
    
    kind: ConfigMap
    apiVersion: v1
    metadata:
      name: nginx-configuration
      namespace: ingress-nginx
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
    
    ---
    kind: ConfigMap
    apiVersion: v1
    metadata:
      name: tcp-services
      namespace: ingress-nginx
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
    
    ---
    kind: ConfigMap
    apiVersion: v1
    metadata:
      name: udp-services
      namespace: ingress-nginx
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
    
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: nginx-ingress-serviceaccount
      namespace: ingress-nginx
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
    
    ---
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: ClusterRole
    metadata:
      name: nginx-ingress-clusterrole
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
    rules:
      - apiGroups:
          - ""
        resources:
          - configmaps
          - endpoints
          - nodes
          - pods
          - secrets
        verbs:
          - list
          - watch
      - apiGroups:
          - ""
        resources:
          - nodes
        verbs:
          - get
      - apiGroups:
          - ""
        resources:
          - services
        verbs:
          - get
          - list
          - watch
      - apiGroups:
          - ""
        resources:
          - events
        verbs:
          - create
          - patch
      - apiGroups:
          - "extensions"
          - "networking.k8s.io"
        resources:
          - ingresses
        verbs:
          - get
          - list
          - watch
      - apiGroups:
          - "extensions"
          - "networking.k8s.io"
        resources:
          - ingresses/status
        verbs:
          - update
    
    ---
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: Role
    metadata:
      name: nginx-ingress-role
      namespace: ingress-nginx
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
    rules:
      - apiGroups:
          - ""
        resources:
          - configmaps
          - pods
          - secrets
          - namespaces
        verbs:
          - get
      - apiGroups:
          - ""
        resources:
          - configmaps
        resourceNames:
          # Defaults to "<election-id>-<ingress-class>"
          # Here: "<ingress-controller-leader>-<nginx>"
          # This has to be adapted if you change either parameter
          # when launching the nginx-ingress-controller.
          - "ingress-controller-leader-nginx"
        verbs:
          - get
          - update
      - apiGroups:
          - ""
        resources:
          - configmaps
        verbs:
          - create
      - apiGroups:
          - ""
        resources:
          - endpoints
        verbs:
          - get
    
    ---
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: RoleBinding
    metadata:
      name: nginx-ingress-role-nisa-binding
      namespace: ingress-nginx
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: nginx-ingress-role
    subjects:
      - kind: ServiceAccount
        name: nginx-ingress-serviceaccount
        namespace: ingress-nginx
    
    ---
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: ClusterRoleBinding
    metadata:
      name: nginx-ingress-clusterrole-nisa-binding
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: nginx-ingress-clusterrole
    subjects:
      - kind: ServiceAccount
        name: nginx-ingress-serviceaccount
        namespace: ingress-nginx
    
    ---
    
    apiVersion: apps/v1
    # kind: Deployment
    kind: DaemonSet
    metadata:
      name: nginx-ingress-controller
      namespace: ingress-nginx
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
    spec:
      # 删除Replicas
      # replicas: 1
      selector:
        matchLabels:
          app.kubernetes.io/name: ingress-nginx
          app.kubernetes.io/part-of: ingress-nginx
      template:
        metadata:
          labels:
            app.kubernetes.io/name: ingress-nginx
            app.kubernetes.io/part-of: ingress-nginx
          annotations:
            prometheus.io/port: "10254"
            prometheus.io/scrape: "true"
        spec:
          # wait up to five minutes for the drain of connections
          terminationGracePeriodSeconds: 300
          serviceAccountName: nginx-ingress-serviceaccount
          # 选择对应标签的node
          nodeSelector:
            isIngress: "true"
            # kubernetes.io/os: linux
          # 使用hostNetwork暴露服务
          hostNetwork: true
          containers:
            - name: nginx-ingress-controller
              image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:master
              args:
                - /nginx-ingress-controller
                - --configmap=$(POD_NAMESPACE)/nginx-configuration
                - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
                - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
                - --publish-service=$(POD_NAMESPACE)/ingress-nginx
                - --annotations-prefix=nginx.ingress.kubernetes.io
              securityContext:
                allowPrivilegeEscalation: true
                capabilities:
                  drop:
                    - ALL
                  add:
                    - NET_BIND_SERVICE
                # www-data -> 101
                runAsUser: 101
              env:
                - name: POD_NAME
                  valueFrom:
                    fieldRef:
                      fieldPath: metadata.name
                - name: POD_NAMESPACE
                  valueFrom:
                    fieldRef:
                      fieldPath: metadata.namespace
              ports:
                - name: http
                  containerPort: 80
                  protocol: TCP
                - name: https
                  containerPort: 443
                  protocol: TCP
              livenessProbe:
                failureThreshold: 3
                httpGet:
                  path: /healthz
                  port: 10254
                  scheme: HTTP
                initialDelaySeconds: 10
                periodSeconds: 10
                successThreshold: 1
                timeoutSeconds: 10
              readinessProbe:
                failureThreshold: 3
                httpGet:
                  path: /healthz
                  port: 10254
                  scheme: HTTP
                periodSeconds: 10
                successThreshold: 1
                timeoutSeconds: 10
              lifecycle:
                preStop:
                  exec:
                    command:
                      - /wait-shutdown
    
    ---
    
    apiVersion: v1
    kind: LimitRange
    metadata:
      name: ingress-nginx
      namespace: ingress-nginx
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
    spec:
      limits:
      - default:
        min:
          memory: 90Mi
          cpu: 100m
        type: Container
</details>  

部署成功  
```
[root@vm-node1 ingress]# kubectl get pods -n ingress-nginx
NAME                             READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-z9f75   1/1     Running   0          2d
```

##### 2. 创建与dashboard service关联的k8s ingress资源  
因为dashboard是使用https进行访问的，这里创建ingress资源之前，首先要准备一下ssl证书，先用自签名证书来代替  

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout kube-dashboard.key -out kube-dashboard.crt -subj "/CN=dashboard.kube.com/O=dashboard.kube.com"
```

使用生成的证书创建 k8s Secret 资源，下一步创建的 Ingress 会引用这个 Secret：  
```
kubectl create secret tls kube-dashboard-ssl --key kube-dashboard.key --cert kube-dashboard.crt -n kube-system
```

创建 Ingress 资源对象（支持 HTTPS 访问）：
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: dashboard-ingress
  namespace: kubernetes-dashboard
  annotations:
    nginx.ingress.kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  tls:
  - hosts:
    - dashboard.kube.com
    secretName: kube-dashboard-ssl
  rules:
    - host: dashboard.kube.com
      http:
        paths:
        - path: /
          backend:
            serviceName: kubernetes-dashboard
            servicePort: 443
```
> 注意：创建的 Ingress 必须要和对外暴露的 Service 在同一命名空间下！ 

##### 3. 把域名和nodeIp配置在客户端的hosts文件里  
在集群外面通过域名访问dashboard，只要修改一下hosts文件，把域名和ingress Controller所在的ip绑定一下就好  
```
# vi /etc/hosts
dashboard.kube.com 172.2.2.1
```

然后就可以以访问了

#### 参考  
1. [k8s ingress原理及ingress-nginx部署测试](https://segmentfault.com/a/1190000019908991)
2. [使用-Kubernetes-Ingress-对外暴露服务](https://qhh.me/2019/08/12/%E4%BD%BF%E7%94%A8-Kubernetes-Ingress-%E5%AF%B9%E5%A4%96%E6%9A%B4%E9%9C%B2%E6%9C%8D%E5%8A%A1/)  