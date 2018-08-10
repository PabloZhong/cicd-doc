# 通过Helm搭建Jenkins
本文档主要介绍如何在EKS容器云平台中通过helm搭建Jenkins，并完成必要的配置。
## Jenkins代码仓库部署与配置
### Step 1: 上传Jenkins与Jenkins-slave镜像至EKS平台的镜像仓库   
  
```
[root@docker ~]# docker login -u 2a9854ed9dd84ef4 -p Ta2380c5364deeca7 172.16.0.176
Login Succeeded
[root@docker ~]# docker pull jenkinsci/blueocean:1.5.0
[root@docker ~]# docker tag jenkinsci/blueocean:1.5.0  172.16.0.176/2a9854ed9dd84ef4/jenkinsci/blueocean:1.5.0
[root@docker ~]# docker push 172.16.0.176/2a9854ed9dd84ef4/jenkinsci/blueocean:1.5.0

[root@docker ~]# docker pull jenkins/jnlp-slave:3.10-1
[root@docker ~]# docker tag jenkins/jnlp-slave:3.10-1 172.16.0.176/2a9854ed9dd84ef4/jenkins/jnlp-slave:3.10-1
[root@docker ~]# docker push  172.16.0.176/2a9854ed9dd84ef4/jenkins/jnlp-slave:3.10-1
```
注：Jenkins BlueOcean镜像使用指南可参考 https://jenkins.io/doc/book/installing/#downloading-and-running-jenkins-in-docker
可以在EKS界面查看已上传至镜像仓库的Jenkins和Jenkins-slave镜像，接下来会基于它们来部署Jenkins应用。
![](https://github.com/ylcao/CICD/blob/master/Images/jenkins-image.png?raw=true)

### Step 2: 在EKS容器云平台中通过Helm部署Jenkins
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

#### (2)下载Jenkins helm安装包，并进行配置

在K8S主节点中下载GitLab helm：
```
wget https://github.com/ylcao/CICD/blob/master/helm/jenkins.tar.gz
tar zxf jenkins.tar.gz
```
配置jenkins helm 主values.yaml;根据环境情况，主要修改配置如下(其他根据实际需求进行修改)：


```
Master:
  Name: jenkins-master
  Image: "hub.easystack.io/2a9854ed9dd84ef4/jenkinsci/blueocean"
  ImageTag: "1.5.0"
  ImagePullPolicy: "Always"
# ImagePullSecret: jenkins
  Component: "jenkins-master"
  UseSecurity: true
  AdminUser: admin
  # AdminPassword: <defaults to random>
  AdminPassword: "admin"
  
  
 ServiceType: NodePort

  InstallPlugins:
    - kubernetes
    - workflow-aggregator
    - workflow-job
    - credentials-binding
    - git
    - docker-build-step
    - gitlab-plugin
    - gitlab-hook
Agent:
  Enabled: true
  Image: hub.easystack.io/2a9854ed9dd84ef4/jenkins/jnlp-slave
  ImageTag: 3.10-1

```
#### (3)安装Jenkins


```
 cd jenkins
 helm install -n jenkins -f values.yaml .  --timeout 3600
```
安装成功：

![](https://github.com/ylcao/CICD/blob/master/Images/jenkins-installed.png?raw=true)

### Step 2: 配置Jenkins
##### 首次登陆Jenkins
通过Web浏览器访问http://<EKS任意Node的公网IP:Nodeport>，进入Jenkins界面，对于本文档示例即可访问http://172.16.6.99:31480/
![](https://github.com/ylcao/CICD/blob/master/Images/jenkins-login.png?raw=true)

> 注意：在helm中已经配置了admin的密码，安装了插件，基于kuberntes的slave也已经配置好，所以只需要配置Kubernetes URL。

Kubernetes URL查看方法，在ECS云平台UI界面-【网络资源】-【负载均衡】，找到Kubernetes集群ApiServer对应的LB的公网IP，并使用默认的6443端口

在【系统管理】-【系统设置】-【新增一个云】-【Kubernetes】中，完成Jenkins与Kubernetes相关的配置，下图为示例配置：
![](https://github.com/ylcao/CICD/blob/master/Images/jenkins-kuberntes.png?raw=true)
### Step 3: 验证Jenkins Pipeline

完成以上配置后，您可以在Jenkins中创建一个最简单的“hello world” Pipeline进行验证： 
![](https://github.com/ylcao/CICD/blob/master/Images/jenkins-pipeline.png?raw=true)

脚本如下：

```
podTemplate(label: 'test', cloud: 'kubernetes') {
    node('test') {
        stage('Run shell') {
            sh 'echo hello world'
        }
    }
}
```
保存流水线配置，随后点击“立即构建”开始执行任务构建：
![](https://github.com/ylcao/CICD/blob/master/Images/jenkins-build.png?raw=true)
> 注：在“hello world”这个Pipeline过程中，Jenkins后台将会自动拉取helm配置的salve镜像hub.easystack.io/2a9854ed9dd84ef4/jenkins/jnlp-slave。

可以在EKS界面中观察到后端自动创建的作为Jenkins Slave的Pod：
![](https://github.com/ylcao/CICD/blob/master/Images/jenkins-slave.png?raw=true)
![](https://github.com/ylcao/CICD/blob/master/Images/jenkins-eks-slave.png?raw=true)
![](https://github.com/ylcao/CICD/blob/master/Images/jenkins-output.png?raw=true)
