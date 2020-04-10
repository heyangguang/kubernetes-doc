# Kubernetes Dashboard插件部署

kubernetes Dashboard是k8s集群中的一个WEB管理工具，代码托管在github上。

[点击这里]: https://github.com/kubernetes/dashboard



## Kubernetes Dashboard部署

v1.16.2版本的集群需要安装最新的 2.0+ 版本的Dshboard：

```bash
# 推荐使用下面的这种方式部署
# 下载这个yaml需要科学上网，如果不能下载，请访问我的github👇
# https://github.com/heyangguang/kubernetes-doc/blob/master/yaml/03-install-k8s-dashboard/recommended.yaml
[root@k8s-master-1 ~]# wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta5/aio/deploy/recommended.yaml

# 修改Service为NodePort类型，让我们部署以后可以访问
[root@k8s-master-1 ~]# vim recommended.yaml
.....
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
  type: NodePort 
.....  
```

> 在 YAML 文件中可以看到新版本 Dashboard 集成了一个 metrics-scraper 的组件，可以通过 Kubernetes 的 Metrics API 收集一些基础资源的监控信息，并在 web 页面上展示，所以要想在页面上展示监控信息就需要提供 Metrics API，比如安装 Metrics Server。

直接创建：

```bash
[root@k8s-master-1 ~]# kubectl apply -f recommended.yaml
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
```

新版本的 Dashboard 会被默认安装在 kubernetes-dashboard 这个命名空间下面：

```bash
[root@k8s-master-1 ~]# kubectl get pods -n kubernetes-dashboard
NAME                                         READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-76585494d8-bnxw8   1/1     Running   0          119s
kubernetes-dashboard-6b86b44f87-hfpng        1/1     Running   0          119s
[root@k8s-master-1 ~]# kubectl get svc -n kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.105.230.99    <none>        8000/TCP        17s
kubernetes-dashboard        NodePort    10.103.203.215   <none>        443:32311/TCP   17s
```

我们可以通过上面的`32311`端口去访问Dashboard，要记住使用https，Chrome访问不了的可以使用Firefox测试：

![image-20200410111210389](03-Kubernetes Dashboard-给朋友穿上一件酷酷的外套.assets/image-20200410111210389.png)

![image-20200410111229970](03-Kubernetes Dashboard-给朋友穿上一件酷酷的外套.assets/image-20200410111229970.png)



我们发现登陆方式有两种，一个kubeconfig另一个是token，这里我们创建一个具有全局所有权限的用户来通过token登陆Dashboard（admin-token.yaml）：

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: admin
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: admin
  namespace: kubernetes-dashboard

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin
  namespace: kubernetes-dashboard
```

创建：

```bash
[root@k8s-master-1 ~]# kubectl apply -f admin-token.yaml
clusterrolebinding.rbac.authorization.k8s.io/admin created
serviceaccount/admin created
[root@k8s-master-1 ~]# kubectl get secret -n kubernetes-dashboard | grep "admin-token"
admin-token-drzxk                  kubernetes.io/service-account-token   3      36s
[root@k8s-master-1 ~]# kubectl get secret admin-token-drzxk -o jsonpath={.data.token} -n kubernetes-dashboard |base64 -d
eyJhbGciOiJSUzI1NiIsImtpZCI6ImkwaG56OGZ6NWJwN21vWFZDNGhFazg2dWNBYVlua2xlVVNZUVdpMXRlOE0ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi10b2tlbi1kcnp4ayIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJhZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjczZGY4MWVjLWQ1NzAtNDg1MC1hMGQwLWMzNTY0MGUzOTNiMyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDphZG1pbiJ9.AuzM-msKFZZ-41Ow2xSpV-u3sACPcHUlGvc6NrO4V13p9QFr3uc5z4oGWZU0TBQ5PwPzm4ZEX6UL0AXS8NyBZWrrmUi6Ioz_X9GMtn6KyrUh2Zny4C0Susq8gFO1XYilTx2GTKNbB8c-FCeIZ61-bMDz6L6iTsRbCr8Bwr7ht7kftJt8G4vic4CCEY_rT_ygI5vSP3NnxDHtht3lvF3R1XWx25gIgkSOC4ukoH07zAWJSDDRF7VbEKVL4k5wEVOYGIfznqS_2GkvivQiQt8fn8LDb1JTv1oygS5Cmvv-OfsLDe6O4-1uU6cUEL5uz0187VMp5-UaQLqnXHddvwxLXg
```

然后使用上面的base64解码后的字符串作为token登陆Dashboard即可:

![image-20200410111640566](03-Kubernetes Dashboard-给朋友穿上一件酷酷的外套.assets/image-20200410111640566.png)

![image-20200410111657904](03-Kubernetes Dashboard-给朋友穿上一件酷酷的外套.assets/image-20200410111657904.png)



当当当！登陆成功，到这个页面我们就算是完成了使用kubeadm搭建v1.16.2单机版本Kubernetes集群的搭建。

最新的Dashboard还有一个黑暗模式哟！

![image-20200410111855972](03-Kubernetes Dashboard-给朋友穿上一件酷酷的外套.assets/image-20200410111855972.png)





## Kubernetes环境重置清理

当然有小伙伴跟我说，在部署Kubernetes集群的时候，因为不可抗力因素，导致部署失败，而环境就乱了。没办法再次重新部署，这个我在生产环境中经常遇到，下面我放出一些命令可以进行系统的重置：

```bash
kubeadm reset
systemctl stop kubelet ; systemctl stop docker
rm -rf /var/lib/cni/
rm -rf /var/lib/kubelet/*
rm -rf /etc/cni/
rm -rf /var/lib/etcd/
rm -rf /etc/kubernetes
ifconfig cni0 down
ifconfig flannel.1 down
ifconfig docker0 down
ip link delete cni0
ip link delete flannel.1
systemctl start docker
```







