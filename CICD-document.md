# CI/CD场景实践操作指南 #
CICD场景实践的开源技术工具链暂定业界比较主流通用、具备代表性的Git+Jenkins+Spinnaker+Harbor+Helm，底层基于ECS 4.0.2和EKS 4.0.2。  

## 1	编写目的 ##
预期读者为：系统实施人员、系统管理人员、系统运行维护人员等。  

## 2	CI/CD说明 ##
## 3	目标 ##

1.	Gitlab 与 Jenkins集成，实现 git push 提交代码，业务自动上线运行，无需人工干预安装过程。
2.	JenKins 与 Maven集成，实现项目代码自动编译。
3.	Jenkins与Docker进行集成，实现镜像自动编译、和发布到Harbor。
4.	Jenkins与 Spinnaker集成，实现Spinnaker管理CI等Jenkins流程
5.	Spinnaker与kubernetes 进行集成，进行分布式构建任务，实现应用自动发布。
基于以上工具链完成Dubbo的应用的编译、打包、部署这一整套CICD流程。

## 4	场景描述 ##
在该场景里面采用ECS里面的大数据组件来实现zookeeper集群或者容器的快速部署，提供dubbo应用架构的服务注册中心，采用EKS来部署dubbo应用，dubbo应用分为两类，一类是提供服务的provider，另一类是消费服务的consumer，两类服务均采用容器部署的方式部署。  

## 5	环境说明 ##
1.	GItlab通过容器进行部署，采用docker.io/library/gitlab: 9.5.3-ce.0镜像
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

## 6 操作流程 ##  
### 6.1	Zookeeper安装部署 ###  

### 6.2 Gitlab安装部署 ###  
Step 1: 将GitLab镜像下载到本地，并上传到EKS平台的镜像仓库中
![](https://note.youdao.com/yws/public/resource/6f3a219a66cbaa0900ebd4ad5d7435e0/xmlnote/701D344AE5D74599ABA0F01747CACA83/1474)



Step 2: 在容器镜像仓库中查看上传的gitlab镜像
![](https://note.youdao.com/yws/public/resource/6f3a219a66cbaa0900ebd4ad5d7435e0/xmlnote/BAC9A60DB6854E539FFF001874963E1B/1477)


Step 3: 在容器平台上部署gitlab服务  

点击创建应用：  
![](https://note.youdao.com/yws/public/resource/6f3a219a66cbaa0900ebd4ad5d7435e0/xmlnote/4BC6450B6D49412783860873A1B00E60/1479)


点击“镜像仓库”，开始通过界面创建：  
![](https://note.youdao.com/yws/public/resource/6f3a219a66cbaa0900ebd4ad5d7435e0/xmlnote/77BF8664344D4A258EB33B6269E9B36B/1481)  


填写Gitlab应用的各项部署参数：  
![](https://note.youdao.com/yws/public/resource/6f3a219a66cbaa0900ebd4ad5d7435e0/xmlnote/278C5DF0E18546D091439B2A4EDB9D2D/1484)  


注意需要配置持久化存储：  

![](https://note.youdao.com/yws/public/resource/6f3a219a66cbaa0900ebd4ad5d7435e0/xmlnote/7562CBD3E1CC40F7AFB219FF0119A2C4/1486)  


### 6.3	项目配置 ###
Step 1: 创建Gitlab项目dubbo，导入dubbo项目：  
从github上将dubbo项目clone下来：git clone https://github.com/ylcao/dubbo.git  
往创建的容器平台的gitlab上push dubbo项目：  
git init  
git remote add origin http://172.16.4.117:31000/zhongyucheng/dubbo.git  
git add .  
touch README.md  
git add README.md  
git commit -m "add README"  
git push -u origin master  

注意：如果在git push过程中一直去寻找旧的https://github.com/ylcao/dubbo.git 地址，需要将.git下面的文件清空即可。

最终文件包括：  
![](https://note.youdao.com/yws/public/resource/6f3a219a66cbaa0900ebd4ad5d7435e0/xmlnote/DCE502B6E92642B98C6518110F30EB88/1488)  


Step 2：修改Dubbo配置文件

（1）dubbo/dubbo-demo/dubbo-demo-consumer/src/main/assembly/conf/dubbo.properties

dubbo.registry.address=zookeeper://172.16.2.245:2181

（2）dubbo/dubbo-demo/dubbo-demo-provider/src/main/assembly/conf/dubbo.properties
dubbo.registry.address=zookeeper://172.16.2.245:2181

# 5	Jenkins Docker Build配置 #
## 5.1	虚拟机上Docker安装(略) ##
在虚拟机上安装Docker,并部署jenkins.
## 5.2	虚拟机上DockerBuild启用 ##
(用于jenkins的Docker插件调用)
step 1:安装略 
step 2:配置：
       (1)docker daemon的DOCKER_OPTS中配置“--insecure-registry 172.16.0.176”参数.
          修改/usr/lib/systemd/system/docker.service中的ExecStart=/usr/bin/dockerd --insecure-registry 172.16.0.176 $OPTIONS \
       (2)修改/usr/lib/systemd/system/docker.service 为：
          ExecStart=/usr/bin/dockerd-current -H tcp://0.0.0.0:4243 -H unix:///var/run/docker.sock \  
step 3:重启docker服务





