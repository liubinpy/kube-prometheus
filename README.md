# Kube-prometheus
[Prometheus operator](https://prometheus-operator.dev/)为Kubernetes提供Prometheus的本地部署和管理以及相关的监控组件。这个项目的目的是简化和自动化基于Prometheus的Kubernetes集群监控配置。

## 组建
- prometheus-operator（v0.47.0）
- Highly available prometheus (v2.26.0)
- Highly available alertmanager (v0.21.0)
- node-exporter (v1.1.2)
- prometheus-adapter (v0.8.4)
- kube-state-metrics (v2.0.0)
- grafana (7.5.4)
- blackbox-exporter (v0.18.0)

## 版本说明
版本说明：基于kube-prometheus release-0.8分支，修改了以下内容：
删除了coredns的监控，添加了kube-dns的监控
修改了kube-controller-manager的servicemonitor，添加了controller-manager的service，修改了kube-controller-manager的dashboard
修改了kube-scheduler的servicemonitor，添加了kube-scheduler的service
添加了grafan的持久化(PVC)和prometheus的持久化(StorageClass)
修改了镜像仓库地址，添加了imagePullSecrets
修改了qa和dev环境的grafana、prometheus的副本数为1，生产环境建议设置为3
添加了nacos的监控和dashboard
添加了springboot的监控，基于kuberntes的服务发现
添加了对名称空间koderover-dev-open-store、public-service-env-qa的rbac权限（需要配置其他名称空间的rbac权限，可参考README）2021-5-8 
添加对nginx-controller的监控和dashboard 2021-5-10 
修改了promethues的数据保留时间为30d 2021-5-20

## 配置
### 持久化
修改grafana-deployment.yaml，需要手动的去创建一个pvc，挂载到/var/lib/grafana下。
grafana-deployment.yaml
```yaml
...
        volumeMounts:
        - mountPath: /var/lib/grafana
          name: grafana-storage
          readOnly: false
...
      volumes:
      - name: grafana-storage
        persistentVolumeClaim:
          claimName: grafana-pvc
```

修改prometheus-prometheus.yaml，添加storage的配置，需要提前准备好StorageClass。
prometheus-prometheus.yaml
```yaml
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: public
        resources:
          requests:
            storage: 500Gi
```

## CRD说明
- `Prometheus`：声明 Prometheus 资源对象期望的状态，Operator 确保这个资源对象运行时一直与定义保持一致
- `Alertmanager`：它定义了所需的Alertmanager的deployment
- `ThanosRuler`：它定义了thanos ruler，默认不使用，需要结合Thanos使用
- `ServiceMonitor`：它以声明的方式指定应该如何监视Kubernetes service。Operator根据API服务器中对象的当前状态自动生成Prometheus scrape配置
- `PodMonitor`：它以声明的方式指定应该如何监视一组pods。Operator根据API服务器中对象的当前状态自动生成Prometheus scrape配置
- `Probe`：它以声明的方式指定应该如何监视一组入口或静态目标。Operator根据定义自动生成Prometheus scrape配置
- `PrometheusRule`：它定义了一组所需的普罗米修斯警报和/或记录规则。Operator生成一个规则文件，Prometheus实例可以使用该文件
- `AlertmanagerConfig`：以声明的方式指定Alertmanager配置部分，允许将警报路由到自定义接收者，并设置报警抑制规则


## 安装
```bash
$ cd kube-prometheus
$ kubectl apply -f setup/    # 安装crd资源，prometheus-operator
$ kubectl apply -f custom/   # 安装自定义的servicemonitor
$ cd ..
$ kubectl apply -f kube-prometheus/ # 安装alertmanager、blackbox、grafana、kube-state-metrics、node-exporter、prometheus-adapter、servicemonitor、rbac等
```


## 注意
默认情况下，`prometheus`的`serviceAccount=prometheus-k8s`只能对`monitoring`、`default`、`kube-system`namespace中roleBinding限制的资源进行访问（主要是服务发现时使用），如果需要访问其他名称空间的资源，需要修改`prometheus-roleSpecificNamespaces.yaml`和`prometheus-roleBindingSpecificNamespaces.yaml`，举例如下：
添加对`koderover-dev-open-store`名称空间的权限：
`prometheus-roleSpecificNamespaces.yaml`中为`RoleList`类型的资源，只需要添加item即可。
```yaml
- apiVersion: rbac.authorization.k8s.io/v1 
  kind: Role
  metadata:
    labels:
      app.kubernetes.io/component: prometheus
      app.kubernetes.io/name: prometheus
      app.kubernetes.io/part-of: kube-prometheus
      app.kubernetes.io/version: 2.26.0
    name: prometheus-k8s
    namespace: koderover-dev-open-store
  rules:
  - apiGroups:
    - ""
    resources:
    - services
    - endpoints
    - pods
    verbs:
    - get
    - list
    - watch
  - apiGroups:
    - extensions
    resources:
    - ingresses
    verbs:
    - get
    - list
    - watch
  - apiGroups:
    - networking.k8s.io
    resources:
    - ingresses
    verbs:
    - get
    - list
    - watch
```

`prometheus-roleBindingSpecificNamespaces.yaml`中为`RoleBindingList`类型的资源，只需要添加item即可。
```yaml
- apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    labels:
      app.kubernetes.io/component: prometheus
      app.kubernetes.io/name: prometheus
      app.kubernetes.io/part-of: kube-prometheus
      app.kubernetes.io/version: 2.26.0
    name: prometheus-k8s
    namespace: koderover-dev-open-store
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: prometheus-k8s
  subjects:
  - kind: ServiceAccount
    name: prometheus-k8s
    namespace: monitoring
```

