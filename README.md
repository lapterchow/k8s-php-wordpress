# 先决条件

配置好nfs或者ceph共享
```bash
$ cd nfs
$ kubectl apply -f ./
```

## 架构介绍

![avatar](image/php-wordress.jpg)

1.redis 一主两从

2.mariadb 一主两从

3.php 3集群

4.nginx 3集群

5.nfs 共享挂在

6.jenkins、gitlab 宿主机各一

```bash
[root@docker-k8s-01 php-wordpress]# kubectl get pods -n devops
NAME                                      READY     STATUS    RESTARTS   AGE
jenkins-6d679fb875-7spgt                  1/1       Running   0          4d
mariadb-cluster-mariadb-master-0          2/2       Running   0          1h
mariadb-cluster-mariadb-slave-0           2/2       Running   0          1h
mariadb-cluster-mariadb-slave-1           2/2       Running   0          1h
nfs-client-provisioner-768fb9786d-k6xvf   1/1       Running   1          16d
nginx-wordpress-7f99cfb654-6p4x4          1/1       Running   0          1h
nginx-wordpress-7f99cfb654-dx6dk          1/1       Running   0          1h
nginx-wordpress-7f99cfb654-rl7sl          1/1       Running   0          1h
php-wordpress-5fc7cbb7cf-2ctlq            1/1       Running   0          1h
php-wordpress-5fc7cbb7cf-c7m9j            1/1       Running   0          1h
php-wordpress-5fc7cbb7cf-wcbq4            1/1       Running   0          1h
redis-ha-redis-ha-server-0                2/2       Running   0          1h
redis-ha-redis-ha-server-1                2/2       Running   0          1h
redis-ha-redis-ha-server-2                2/2       Running   0          1h

```

## 执行步骤

1.创建devops namespace
```
$ kubectl -f devops-namespace.yaml
```

2.创建redis-ha
[参考](https://github.com/helm/charts/tree/master/stable/redis-ha)
```bash
$ cd redis-ha/
$ helm install -f values.yaml --name redis-ha --namespace devops stable/redis-ha
```

3.创建mariadb
[参考](https://github.com/helm/charts/tree/master/stable/mariadb)
```bash
$ cd mariadb/
$ helm install --name mariadb-cluster --namespace devops -f values-production.yaml stable/mariadb
```

4.创建php
```bash
$ cd php/
$ kubectl apply -f ./
```

5.创建nginx
```bash
$ cd nginx/
$ kubectl apply -f ./
```

## 其他点

1.jenkins使用宿主机或者容器,下面是使用容器的方式,本实验使用宿主机,更新代码之后直接同步到nfs or ceph,如果nginx,php在同一容器可使用jenkins容器将代码直接坐进容器中
```
$ cd jenkins
$ helm install ./ --values values.yaml --name=jenkins --namespace=devops
```

2.gitalb使用宿主机,没有现成比较好的解决方案,gitlab设计比较多的依赖: redis、数据库等


## 配置自动触发CI
+ 配置Gitlab项目
在Gitlab中创建一个测试项目，将上面测试的脚本内容写入到一个Jenkinsfile文件中，然后上传到该测试项目根路径下。

+ 配置Jenkins项目
点击项目配置——>构建触发器——>勾选Build when a change is pushed to GitLab. GitLab webhook URL:http://jenkins.local.com/project/k8s-test——>保存配置

+ 配置Webhook
进入Gitlab测试项目的Settings——>Integrations，一般只需要填写URL即可，其他的可根据需求环境配置 默认Jenkins配置不允许匿名用户触发构建，因此还需要添加用户和token。
URL的格式为：
http://[UserID]:[API Token]@jenkins.local.com/project/[ProjectName]

Jenkins 用户ID Token查看： 点击右上角的用户名——>设置——>API Token(点击Show API Token...)

最终Webhook中的URL类似： http://admin:a910b1492e39e9dd1ea48ea7f7638aaf@jenkins.local.com/project/k8s-test

后面只需要我们一提交代码到Git仓库，就会自动触发Jenkins进行构建了。


# 参考链接
[参考](https://github.com/gjmzj/kubeasz/blob/master/docs/guide/jenkins.md#%E9%85%8D%E7%BD%AE%E8%87%AA%E5%8A%A8%E8%A7%A6%E5%8F%91ci)
[参考](https://www.kancloud.cn/huyipow/kubernetes/722822)
