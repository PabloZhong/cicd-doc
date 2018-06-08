# CI/CD场景实践操作指南
## 概述
阐述文档及设计场景的目的：  
CICD场景实践的开源技术工具链暂定业界比较主流通用、具备代表性的Git+Jenkins+Spinnaker+Harbor+Helm，底层基于ECS 4.0.2和EKS 4.0.2。  

## 场景描述
1.搭建CICD工具链：Git+Jenkins+Spinnaker+Harbor+Helm  
  两种方式构建：1）EKS平台直接通过界面操作；  
  2）EKS+Helm实现，为进入应用中心做准备。    
2.Dubbo微服务Demo代码的编译、打包、并且在EKS上部署成功，能够CICD流程打通。   


在该场景里面采用ECS里面的大数据组件来实现zookeeper集群或者容器的快速部署，提供dubbo应用架构的服务注册中心，采用EKS来部署dubbo应用，dubbo应用分为两类，一类是提供服务的provider，另一类是消费服务的consumer，两类服务均采用容器部署的方式部署。  

1.	Gitlab 与 Jenkins集成，实现 git push 提交代码，业务自动上线运行，无需人工干预安装过程。
2.	JenKins 与 Maven集成，实现项目代码自动编译。
3.	Jenkins与Docker进行集成，实现镜像自动编译、和发布到Harbor。
4.	Jenkins与 Spinnaker集成，实现Spinnaker管理CI等Jenkins流程
5.	Spinnaker与kubernetes 进行集成，进行分布式构建任务，实现应用自动发布。
基于以上工具链完成Dubbo的应用的编译、打包、部署这一整套CICD流程。  

## 目标  
掌握cicd工具链的搭建，和基本的cicd操作流程。  

## 环境说明（描述不准确）
1.	GitLab在EKS中进行部署，采用docker.io/library/gitlab: 9.5.3-ce.0镜像
2.	Jenkins通过容器进行部署，采用Jenkins:2.46.2
3.	Jenkins 中的Docker build地址通过虚拟机安装Docker服务配置暴露地址
4.	Zookeeper采用容器部署(最新)或者ECS中大数据组件中Zookeeper
5.	Dubbo Demo源代码地址：git@github.com:ylcao/dubbo.git
6.	Spinnaker Helm部署程序地址：https://github.com/ylcao/spinnaker-k8s/tree/master/kubernetes-helm-spinnaker
7.	ESCloud 4.0.2
<table>
   <tr>
      <td>序号</td>
      <td>软件环境</td>
      <td>访问地址</td>
      <td>说明</td>
   </tr>
   <tr>
      <td>1</td>
      <td>GItlab容器</td>
      <td>http://172.16.3.4:31287/</td>
      <td>容器中配置持久化存储和LB</td>
   </tr>
   <tr>
      <td>2</td>
      <td>Jenkins容器</td>
      <td>http://172.16.3.3:32550</td>
      <td>虚拟中安装Jenkins与Docker(用于jenkins的Docker插件调用)</td>
   </tr>
   <tr>
      <td>3</td>
      <td>Harbor</td>
      <td>http://172.16.0.176</td>
      <td></td>
   </tr>
   <tr>
      <td>4</td>
      <td>Kubernetes主节点</td>
      <td>172.16.3.82</td>
      <td>主节点地址或者VIP</td>
   </tr>
   <tr>
      <td>5</td>
      <td>Jenkins Docker build虚拟机</td>
      <td>172.16.1.30</td>
      <td>Jenkins 编译Docker镜像时采用</td>
   </tr>
   <tr>
      <td>6</td>
      <td>Zookeeper容器</td>
      <td>172.16.2.245</td>
      <td></td>
   </tr>
   <tr>
      <td>7</td>
      <td>Spinnaker</td>
      <td>http://172.16.3.151:9000</td>
      <td>地址随机生成</td>
   </tr>
   <tr>
      <td></td>
   </tr>
</table>  

## 操作流程说明  

### 1.在EKS中部署GitLab  

**Step 1: 上传GitLab镜像至EKS平台的镜像仓库。**   

首先需要准备一个安装有单机版Docker CE软件的操作系统环境，可以是本地虚拟机，也可以是ECS平台中的云主机，注意需要能够与EKS镜像仓库的实现网络互通。  

注意：需要配置Docker Daemon的DOCKER_OPTS参数，添加“--insecure-registry x.x.x.x”参数。  
不同操作系统的配置方式略有差异，请以Docker官方说明为准。  
以本文档所采用的CentOS 7.2.1511为例，可参考以下配置方法：  
```
[root@docker-ce ~]# vi /usr/lib/systemd/system/docker.service
```
配置参考示例如下：  
![](Images/docker-daemon.png)

然后执行：  
```
[root@docker-ce ~]# systemctl daemon-reload  
[root@docker-ce ~]# systemctl restart docker  
```

尝试登陆镜像仓库，参考EKS界面“本地镜像仓库"-"上传镜像"的步骤说明：  
![](Images/login-registry.png)


提示“Login Succeed”之后，便可以将本地的镜像推送至镜像仓库。  
将所需版本的GitLab镜像下载到本地（需能够访问外网从Dockerhub拉取镜像）：  
```
[root@docker-ce ~]# docker pull gitlab/gitlab-ce:10.7.4-ce.0
```  

修改镜像的Tag，并上传镜像到EKS平台的镜像仓库中:  
```
[root@docker-ce ~]# docker images
[root@docker-ce ~]# docker tag gitlab/gitlab-ce:10.7.4-ce.0  172.16.0.176/3dc70621b8504c98/gitlab-ce:10.7.4-ce.0
[root@docker-ce ~]# docker push 172.16.0.176/3dc70621b8504c98/gitlab-ce:10.7.4-ce.0
```  

注：GitLab镜像使用可参考 https://docs.gitlab.com/omnibus/docker/#run-the-image  

可以在EKS界面查看已上传至镜像仓库的GitLab镜像，接下来会基于它来部署GitLab应用。   

![](Images/check-gitlab-images.png)

**Step 2: 在EKS容器平台上部署GitLab服务。**  

点击"创建应用"，并选择通过"镜像仓库"开始创建。  

![](Images/create-gitlab-1.png)  
![](Images/create-gitlab-2.png)  

填写“应用名称”，然后点击“添加服务”，在弹出框中填入服务的各项配置参数。  
填写“服务名称”，选择上一步所上传的GitLab镜像，填入Pod的基本配置：  
![](Images/gitlab-configuration-1.png)  
注意：  
1）GitLab容器消耗计算资源比较多，因此图示中分配了4Cores/4096MiB计算资源；  
2）需要配置持久化存储，将容器的3个目录/var/opt/gitlab （存储应用数据)、 /var/log/gitlab （存储log文件）、 /etc/gitlab（存储配置文件）挂载出来。  

下一步，填写服务（即Kubernetes Service）访问设置，在这里我们选取NodePort方式，将GitLab容器的3个端口（80、22和443）暴露出来，映射服务端口也设为80、22和443，另外，指定对应的节点暴露端口30080、30022和30443，如图示例：
![](Images/gitlab-configuration-2.png)  

下一步，注入环境变量至GitLab容器中，参考下图：  
![](Images/gitlab-configuration-3.png)  
图示中键填入为： GITLAB_OMNIBUS_CONFIG  
值填入为：  external_url 'http://gitlab.example.org/'; gitlab_rails['gitlab_shell_ssh_port'] = 30022;  
分别代表GitLab的外部访问域名和SSH连接端口，其中外部访问域名还需要在接下来的Ingress路由中设置。  

保存上述配置，便可以部署GitLab应用。可在EKS界面查看已经创建完成的GitLab应用。  
![](Images/gitlab-configuration-4.png)  

此时已经可以通过NodePort方式访问GitLab，但是为了能够通过域名（本示例为gitlab.example.org）访问，我们可以设置路由(Ingress)，提供外部负载均衡访问。  
![](Images/gitlab-configuration-5.png)  
![](Images/gitlab-configuration-6.png)  
注意需要配置DNS域名解析，可采用以下两种方式：  
1）如果环境中有DNS服务器，则直接配置DNS解析即可，例如将上图中的gitlab.example.org映射到Kubernetes集群的某一个Slave节点的公网IP（注意不能为Master节点）；  
2）如果环境中没有DNS服务器，则可以配置本地hosts文件，对Windows而言为C:\Windows\System32\drivers\etc\hosts，对于上图中的示例需要添加一条： 172.16.4.191 gitlab.example.org  

可通过浏览器访问GitLab：  
![](Images/access-to-gitlab.png)  
注册一个新的账号即可正常使用。  


### 2.GitLab项目配置  
**Step 1: 创建Gitlab项目dubbo，导入dubbo源代码。**  
从github上将dubbo项目clone下来：git clone https://github.com/ylcao/dubbo.git  
往创建的容器平台的gitlab上push dubbo项目：  

通过SSH key pair方式访问GitLab可参考：https://docs.gitlab.com/ee/ssh/README.html   

```
git init  
git remote add origin ssh://git@gitlab.example.org:30022/easystack/dubbo.git  
git add .  
touch README.md  
git add README.md  
git commit -m "add README"  
git push -u origin master  
```

注意：如果在git push过程中一直去寻找旧的https://github.com/ylcao/dubbo.git 地址，需要将.git下面的文件清空即可。

最终文件包括：  
![](https://note.youdao.com/yws/public/resource/6f3a219a66cbaa0900ebd4ad5d7435e0/xmlnote/DCE502B6E92642B98C6518110F30EB88/1488)  


Step 2：修改Dubbo配置文件

（1）dubbo/dubbo-demo/dubbo-demo-consumer/src/main/assembly/conf/dubbo.properties

dubbo.registry.address=zookeeper://172.16.2.245:2181

（2）dubbo/dubbo-demo/dubbo-demo-provider/src/main/assembly/conf/dubbo.properties
dubbo.registry.address=zookeeper://172.16.2.245:2181


### 3.Jenkins Docker Build配置 
#### 3.1 安装jenkins 

step 1. 下载jenkins docker镜像：

```
[root@docker-ce ~]#  docker pull jenkins:2.60.3
[root@docker-ce ~]# docker pull jenkinsci/blueocean：1.5.0
```
```
[root@docker-ce /]# docker run -p 8080:8080 -u 0 -p 50000:50000 -v /your/home:/var/jenkins_home jenkins:2.60.3
```
此命令中-u 0 参数是覆盖容器中内置的帐号，该用外部传入，这里传入0代表的是root帐号Id。

step 2. 上传到私有仓库中去：

```
[root@docker-ce ~]# docker tag jenkins:2.60.3 172.16.0.176/3dc70621b8504c98/jenkins
[root@docker-ce ~]# docker push 172.16.0.176/3dc70621b8504c98/jenkins
```
在页面查看镜像仓库中jenkins镜像：

![](Images/check-jenkins-images.png)


step 3. 在EKS平台上部署jenkins服务：

点击右上角创建应用，选择镜像仓库，选择jenkins镜像

![](Images/jenkins-conf1.png)

![](Images\jenkins-conf2.png)


注意：jenkins镜像部署成功后，需要在部署的yaml中添加securityContext来修改访问/var/jenkins_home的用户为user 0，即root用户，具体修改如下：
![](Images/rewrite-yaml.png)

修改完成后，yaml文件会变为：
![](Images/finished-yaml.png)
部署yaml修改完成后，jenkins服务启动正常，在EKS平台查看创建成功的jenkins服务：
![](Images/check-jenkins-service.png)

jenkins Server要想能与k8s集群的apiserver通信，需要先通过权限认证。k8s里面有个Service Account的概念，配置使用Service Account来实现给Jenkins Server的授权。步骤如下：
使用jenkins-rbac.yaml来创建service Account:
```
kubectl create -f jenkins-rbac.yaml
```
```
jenkins-rbac.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-admin-new
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin-new
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jenkins-admin-new
  namespace: default
```
查看生成的“service account”如下：
![](Images/check-serviceaccount.png)

将生成的名为“jenkins-admin-new”的serviceaccount加入jenkins的部署yaml中：
![](Images/add-serviceaccount.png)

部署yaml修改完成后，jenkins服务启动正常.

访问http://172.16.4.190:30601/地址来访问jenkins，首次登陆jenkins，需要输入初始密码：
![](Images/jenkins-initial-password.png)
使用命令：
```
[escore@ci-akyzklrim5-0-vsx3xunzxan2-kube-master-gko2lwdxza5r ~]$ kubectl get pod
NAME                                                READY     STATUS    RESTARTS   AGE
gitlab-cicd-gitlab-cicd-pfcpzdqa-173637270-tm7cp    1/1       Running   0          1d
gitlab-test-gitlab-test-t2oure6b-1164666112-lms0x   1/1       Running   0          20h
jenkins-jenkins-bj3we2kn-1700231787-sq1l2           1/1       Running   0          12h
[escore@ci-akyzklrim5-0-vsx3xunzxan2-kube-master-gko2lwdxza5r ~]$ kubectl exec -it jenkins-jenkins-bj3we2kn-1700231787-sq1l2 bash
bash-4.4# ls

```
查看初始密码，并输入，进入jenkins页面：
![](Images/jenkins-web1.png)

首先修改admin的密码：

![](Images/jenkins-change-password.png)


#### 3.2	Jenkins插件安装
(用于jenkins的Docker插件调用)

step 1:安装以下jenkins插件：

Docker build step plugin

Git plugin

Gitlab Hook Plugin

Maven Integration plugin

进入jenkins【系统管理】页面，选择【管理插件】中选择以上插件，并进行安装:
![](Images/install-jenkinsplugin-1.png)

![](Images/select-jenkinsplugin.png)





