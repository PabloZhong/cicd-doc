# 通过Helm搭建GitLab
本文档主要介绍如何在EKS容器云平台中通过helm搭建基于Redis与Postgresql的GitLab，并完成必要的配置。
## GitLab代码仓库部署与配置
### Step 1: 上传GitLab、Redis与Postgresql镜像至EKS平台的镜像仓库  

首先需要准备一个安装有单机版Docker CE软件的操作系统环境用于上传Docker镜像，可以使用本地虚拟机，也可以使用ECS平台中的云主机，注意需要能够与EKS镜像仓库实现网络互通。  

注意：需要配置Docker Daemon的DOCKER_OPTS参数，添加“--insecure-registry x.x.x.x”参数。  
不同操作系统的配置方式略有差异，请以Docker官方说明为准。  
以本文档所采用的CentOS 7.2.1511为例，可参考以下配置方法：  
```
[root@docker-ce ~]# vi /usr/lib/systemd/system/docker.service
```
配置参考示例如下：  
![](https://github.com/PabloZhong/cicd-doc/raw/master/1_%E5%9F%BA%E7%A1%80%E5%AE%9E%E8%B7%B5/Images/2/docker-daemon.png)

然后执行：  
```
[root@docker-ce ~]# systemctl daemon-reload  
[root@docker-ce ~]# systemctl restart docker  
```

尝试登陆镜像仓库，参考EKS界面“本地镜像仓库"-"上传镜像"的步骤说明：  
![](https://github.com/ylcao/CICD/blob/master/Images/login-registry.png?raw=true)


```
[root@docker ~]# docker login -u 2a9854ed9dd84ef4 -p Ta2380c5364deeca7 172.16.0.176
```

提示“Login Succeed”之后，便可以将本地的镜像推送至镜像仓库。  
#### (1)上传GitLab镜像
首先需将所需版本的GitLab镜像下载到本地（需能够访问外网从Dockerhub拉取镜像）：  

```
[root@docker ~]# docker pull gitlab/gitlab-ce:10.3.7-ce.0
```
修改镜像的Tag，并上传镜像到EKS平台的镜像仓库中:
```
[root@docker ~]# docker tag gitlab/gitlab-ce:10.3.7-ce.0 172.16.0.176/2a9854ed9dd84ef4/gitlab-ce:10.3.7-ce.0
[root@docker ~]# docker push 172.16.0.176/2a9854ed9dd84ef4/gitlab-ce:10.3.7-ce.0
```
#### (2)上传Redis镜像


```
[root@docker ~]# docker pull bitnami/redis:3.2.9-r2
[root@docker ~]# docker tag bitnami/redis:3.2.9-r2 172.16.0.176/2a9854ed9dd84ef4/redis:3.2.9-r2
[root@docker ~]# docker push 172.16.0.176/2a9854ed9dd84ef4/redis:3.2.9-r2
```


#### (3)上传Postgresql镜像


```
[root@docker ~]# docker pull postgres:9.6.2
[root@docker ~]# docker tag postgres:9.6.2 172.16.0.176/2a9854ed9dd84ef4/postgres:9.6.2
[root@docker ~]# docker push 172.16.0.176/2a9854ed9dd84ef4/postgres:9.6.2
```
可以在EKS界面查看已上传至镜像仓库的GitLab、Redis、Postgresql镜像，接下来会基于它们来部署GitLab应用。
![](https://github.com/ylcao/CICD/blob/master/Images/image-check.png?raw=true)
### Step 2: 在EKS容器云平台中通过Helm部署GitLab
需要准备一台虚拟机或者SSH工具，并可以通过ssh登陆到EKS容器云平台
#### (1)安装Helm
登陆到K8S主节点中，安装helm

```
Wget https://kubernetes-helm.storage.googleapis.com/helm-v2.8.2-linux-amd64.tar.gz
tar zxf helm-v2.8.2-linux-amd64.tar.gz
cd linux-amd64/
cp -rf helm /usr/local/bin/
helm init
helm init --tiller-image omio/gcr.io.kubernetes-helm.tiller:v2.8.2 --upgrade

```
Helm验证(helm version)：
![](https://github.com/ylcao/CICD/blob/master/Images/helm-check.png?raw=true)

#### (2)下载GitLab helm安装包，并进行配置

在K8S主节点中下载GitLab helm：
```
wget https://github.com/ylcao/CICD/blob/master/helm/gitlab-ce.tar.gz
tar zxf gitlab-ce.tar.gz
```
配置GitLab helm 主values.yaml;根据环境情况，主要修改配置如下：
```
vim gitlab-ce/values.yaml

#gitlab-ce/values.yaml
image: hub.easystack.io/2a9854ed9dd84ef4/gitlab-ce:10.3.7-ce.0
externalUrl: http://your-domain.com/
gitlabRootPassword: "passw0rd"
serviceType: LoadBalancer
resources:
uirements
  requests:
    memory: 2Gi
    cpu: 500m
  limits:
    memory: 4Gi
    cpu: 2
postgresql:
  # 9.6 is the newest supported version for the GitLab container
  imageTag: "9.6"
  cpu: 1000m
  memory: 1Gi

  postgresUser: gitlab
  postgresPassword: gitlab
  postgresDatabase: gitlab

  persistence:
    size: 10Gi

redis:
  redisPassword: "gitlab"

  resources:
    requests:
      memory: 1Gi

  persistence:
    size: 10Gi

```
配置GitLab helm charts中Redis的values.yaml，修改镜像地址:

```
vim gitlab-ce/charts/redis/values.yaml

#gitlab-ce/charts/redis/values.yaml
image: hub.easystack.io/2a9854ed9dd84ef4/redis:3.2.9-r2
```
配置GitLab helm charts中postgresql的values.yaml，修改镜像地址:


```
vim gitlab-ce/charts/postgresql/values.yaml

#gitlab-ce/charts/postgresql/values.yaml
image: "hub.easystack.io/2a9854ed9dd84ef4/postgres:9.6.2"
```

#### (3)安装GitLab


```
cd gitlab-ce
 helm install -n gitlab -f values.yaml .  --timeout 3600
```
安装成功：

![](https://github.com/ylcao/CICD/blob/master/Images/GitLab-installed.png?raw=true)

EKS界面验证：
![](https://github.com/ylcao/CICD/blob/master/Images/gitlab-service.png?raw=true)
![](https://github.com/ylcao/CICD/blob/master/Images/gitlab-pod.png?raw=true)

GitLab界面验证：
![](https://github.com/ylcao/CICD/blob/master/Images/gitlab-dashbord.png?raw=true)
