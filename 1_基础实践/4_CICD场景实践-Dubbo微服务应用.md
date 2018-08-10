# CI/CD场景实践-Dubbo微服务应用  （待修改） 

本文档主要介绍如何在已完成部署的CI/CD工具链基础上，实现基于Dubbo框架的微服务应用的CI/CD配置和演示。  

## 1. CI/CD配置  

### 1.1 在GitLab中创建项目，并上传源代码  

**Step 1: 在GitLab中创建示例项目。**  
在GitLab中创建一个示例项目（Create a project）：  
![](Images/4/gitlab-create-project-1.png)  

填入项目名称如“dubbo-demo”，选择项目类型（Private或者Public均可）：  
![](Images/4/gitlab-create-project-2.png)  

**Step 2: 上传源代码至GitLab示例项目。**  

首先需要确认所使用的本地虚拟机环境（可使用之前上传Docker镜像的环境）中已经安装了Git，并完成Git Global Config配置。   
然后通过公网从GitHub上将示例项目的源代码克隆（Clone）到本地虚拟机中：  
```
[root@docker-ce ~]# git clone https://github.com/PabloZhong/dubbo-demo.git
```

随后可参考GitLab中界面提示，通过SSH方式将源代码Push到GitLab中（使用[文档2](./2_搭建CICD工具链.md)中的SSH Key）：  
（备注：需要提前在运行Git的虚拟机上修改/etc/hosts文件，加入集群节点NodeIP与域名的映射，如本示例加入172.16.6.48 gitlab.example.org）
```
[root@docker-ce ~]# cd dubbo-demo/
[root@docker-ce dubbo-demo]# git init
[root@docker-ce dubbo-demo]# git remote rename origin old-origin 
[root@docker-ce dubbo-demo]# git remote add origin ssh://git@gitlab.example.org:30022/easystack/dubbo-demo.git  
[root@docker-ce dubbo-demo]# git push -u origin --all   
```
其中GitLab项目URL地址参考：  
![](Images/4/gitlab-ssh-url.png)  

Push成功后即可在GitLab的“dubbo-demo”项目中看到已上传的源代码：  
![](Images/4/gitlab-check-source-code.png)  
其中的Dockerfile和Jenkinsfile后面步骤中都会使用到。


### 1.2 创建Jenkins Pipeline，并部署Dubbo-demo应用    

**Step 1: 制作Jenkins Slave镜像。**  
为了使用Jenkins Slave来执行Pipeline，首先需要制作Jenkins Slave所使用的Docker镜像，并上传至EKS的镜像仓库中。   
具体步骤如下：  

1） 在本地虚拟机Linux环境中，执行：  
```
[root@docker-ce jenkins-slave]# git clone https://github.com/PabloZhong/jenkins-1.git (或者git clone https://github.com/PabloZhong/jenkins.git)  
[root@docker-ce jenkins-slave]# cd jenkins-1/slave-maven/   (或者cd jenkins/slave-maven)
```

可以查看到所需要使用到的Dockerfile如下：  
```
FROM openshift/jenkins-slave-base-centos7

MAINTAINER Ben Parees <bparees@redhat.com>

ENV MAVEN_VERSION=3.3 \
    GRADLE_VERSION=4.2.1 \
    BASH_ENV=/usr/local/bin/scl_enable \
    ENV=/usr/local/bin/scl_enable \
    PROMPT_COMMAND=". /usr/local/bin/scl_enable" \
    PATH=$PATH:/opt/gradle/bin

# Install Maven
RUN INSTALL_PKGS="java-1.8.0-openjdk-devel.x86_64 rh-maven33*" && \
    x86_EXTRA_RPMS=$(if [ "$(uname -m)" == "x86_64" ]; then echo -n java-1.8.0-openjdk-devel.i686 ; fi) && \
    yum install -y centos-release-scl-rh && \
    yum install -y --enablerepo=centosplus $INSTALL_PKGS $x86_EXTRA_RPMS && \
    curl -LOk https://services.gradle.org/distributions/gradle-${GRADLE_VERSION}-bin.zip && \
    unzip gradle-${GRADLE_VERSION}-bin.zip -d /opt && \
    rm -f gradle-${GRADLE_VERSION}-bin.zip && \
    ln -s /opt/gradle-${GRADLE_VERSION} /opt/gradle && \
    # have temporarily removed the validation for java to work around known problem fixed in fedora; jupierce and gmontero are working with
    # the requisit folks to get that addressed ... will switch back to rpm -V $INSTALL_PKGS when that occurs
    rpm -V  rh-maven33 && \
    yum clean all -y && \
    mkdir -p $HOME/.m2 && \
    mkdir -p $HOME/.gradle

# When bash is started non-interactively, to run a shell script, for example it
# looks for this variable and source the content of this file. This will enable
# the SCL for all scripts without need to do 'scl enable'.
ADD contrib/bin/scl_enable /usr/local/bin/scl_enable
ADD contrib/bin/configure-slave /usr/local/bin/configure-slave
ADD ./contrib/settings.xml $HOME/.m2/
ADD ./contrib/init.gradle $HOME/.gradle/

RUN chown -R 1001:0 $HOME && \
    chmod -R g+rw $HOME

USER 1001
```

2）构建Jenkins Slave镜像  
在Dockerfile所在的路径下执行以下命令进行镜像构建： 
```
[root@docker-ce slave-maven]# docker build -t jenkins-slave-maven:v1 .  
```

3）上传Jenkins Slave镜像  
Jenkins Slave镜像制作完成后，使用docker push命令将Jenkins Slave镜像上传到EKS的镜像仓库中。  
```
[root@docker-ce slave-maven]# docker tag jenkins-slave-maven:v1 172.16.0.176/3dc70621b8504c98/jenkins-slave-maven:v1  
[root@docker-ce slave-maven]# docker push 172.16.0.176/3dc70621b8504c98/jenkins-slave-maven:v1  
```
注：请按需修改上述命令行中的镜像仓库地址和用户名。  

上传成功后，可查看已上传的镜像：  
![](Images/4/check-jenkins-slave-image.png) 
后续步骤中会使用这个镜像来执行Jenkins Pipeline。  

**Step 2: 通过Jenkins Blue Ocean创建Jenkins Pipeline。**   
使用Jenkins Blue Ocean能够实现更丰富、更直观的Pipeline功能。  

在Jenkins主界面点击“Open Blue Ocean”进入Blue Ocean操作界面：   
![](Images/3/jenkins-blue-ocean.png)   

点击“创建流水线”：   
![](Images/3/jenkins-create-pipeline-1.png)   

填入GitLab代码仓库对应的项目地址：（注意：SSH的URL中需要将域名改成NodeIP）   
![](Images/3/jenkins-create-pipeline-2.png)  

Jenkins将自动生成SSH Key Pair，需要将SSH公钥添加到GitLab中，添加路径为【GitLab】-【User Setting】-【SSH Keys】：  
![](Images/3/gitlab-ssh-key.png)  

回到Jenkins Blue Ocean界面，点击“创建Pipeline”之后，Jenkins首先将会自动拉取GitLab代码库中的Jenkinsfile，并按照Jenkinsfile执行第一次Pipeline：  
图缺  
![](Images/3/jenkins-initial-pipeline-1.png)  
![](Images/3/jenkins-initial-pipeline-2.png)  

本示例中的Jenkinsfile参考如下：
```
podTemplate(name: 'jnlp', label: 'jnlp', namespace: 'default', cloud: 'kubernetes',
  containers: [
        containerTemplate(
            name: 'jnlp',
            //请按需修改Jenkins Slave镜像名称
            image: 'hub.easystack.io/3dc70621b8504c98/jenkins-slave:v1',
            command: '',
            args: '${computer.jnlpmac} ${computer.name}',
            privileged: true,
            alwaysPullImage: false,
            ttyEnabled: true, 
        ),
  ],
  volumes: [hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock'),
            hostPathVolume(hostPath: '/usr/bin/docker', mountPath: '/usr/bin/docker'),
            hostPathVolume(hostPath: '/usr/bin/docker-current', mountPath: '/usr/bin/docker-current'),
            hostPathVolume(hostPath: '/etc/sysconfig/docker', mountPath: '/etc/sysconfig/docker'),
            hostPathVolume(hostPath: '/usr/bin/kubectl', mountPath: '/usr/bin/kubectl')]
  ) {

  node('jnlp') {
    stage('CICD for Snake Game demo') {
        container('jnlp') {
            stage("Clone source code of Snake game") {
                //请按需修改Git源代码库地址
                //如果是Private项目，参考示例如下（需使用GitLab Access Token）
                sh """
                    git clone http://oauth2:E8azoQ6QSTpmvyzEeJzc@172.16.6.28:30080/easystack/snake-demo.git
                """
                //如果是Public项目，参考示例如下
                //git 'http://172.16.6.28:30080/easystack/snake-demo.git'
            }
                      
            stage('Build & push docker image') {
                //请按需修改镜像仓库的账号和密码，并注意docker build命令中Dockerfile所在路径
                sh """
                    docker login -u 3dc70621b8504c98 -p Tcdf4f05247d79dd7 hub.easystack.io  
                    docker build -t hub.easystack.io/3dc70621b8504c98/snake:v${BUILD_NUMBER} ./snake-demo
                    docker push hub.easystack.io/3dc70621b8504c98/snake:v${BUILD_NUMBER}
                """
            }
            
            //stage('Deploy app to EKS') {
                //请按需修改Deployment名称和Snake镜像名称
                //sh """kubectl set image deployment/snake-demo-snake-demo-cao7ea5d snake-demo-snake-demo-cao7ea5d=hub.easystack.io/3dc70621b8504c98/snake:v${BUILD_NUMBER}"""
            //}
        }
    }
 }
}
```
其中有以下几点需要说明：  

1）```image: 'hub.easystack.io/3dc70621b8504c98/jenkins-slave:v1'```指定之前Step 1中构建的Jenkins Slave镜像。  
2）```stage("Clone source code of Snake game")```将源代码从GitLab中拉取到Jenkins Slave Pod中，具体写法如下：    
   · 如果是Public类型的GitLab项目，直接通过HTTP方式Git clone源代码即可，无需使用用户名+密码或者Access Token；  
   · 如果是Private类型的GitLab项目，则需要使用```用户名+密码```或使用[文档2](./2_搭建CICD工具链.md)中生成的GitLab ```Access Token```，具体格式参考：  

```
    git clone http://<username>:<password>@<GitLab URL>/<username>/<project name>.git  
    或：  
    git clone http://oauth2:<access token>@<GitLab URL>/<username>/<project name>.git  
```

3）下面的命令分别实现登录镜像仓库、构建Snake Demo镜像以及上传镜像：  
```
 stage('Build & push docker image') {
                //请按需修改镜像仓库的账号和密码，并注意docker build命令中Dockerfile所在路径
                sh """
                    docker login -u 3dc70621b8504c98 -p Tcdf4f05247d79dd7 hub.easystack.io
                    docker build -t hub.easystack.io/3dc70621b8504c98/snake:v${BUILD_NUMBER} ./snake-demo
                    docker push hub.easystack.io/3dc70621b8504c98/snake:v${BUILD_NUMBER}
                """
            }
``` 
其中docker build构建镜像步骤，会使用Jenkins Slave从GitLab代码库中拉取的代码中所包含的Dockerfile。    

在Blue Ocean界面中可以查看Pipeline执行进度：   
![](Images/3/check-initial-pipeline.png)  

可以在EKS界面中看到正在执行Pipeline的Jenkins Slave Pod：   
![](Images/3/check-jenkins-slave-pod.png)  

执行完成第一次Pipleline后，可以在EKS的镜像仓库中查看第一次构建并上传的Snake Demo镜像：  
![](Images/3/check-snake-image.png)  

注：按照上面所示的Jenkinsfile执行的Pipeline，第一次构建只会完成Snake Demo镜像构建并上传到EKS镜像仓库，下一步需要手动进行第一次应用部署。  

**Step 3: 在EKS中进行Snake Demo应用的第一次部署。**  
在EKS中，选择第一次执行Pipeline生成的Snake Demo镜像，进行Snake Demo应用部署： 
![](Images/3/create-initial-snake-1.png)  
![](Images/3/create-initial-snake-2.png)  

部署成功之后，查看对应的服务的端口号：  
![](Images/3/check-snake-service.png)  

通过NodeIP:Port方式，通过Web浏览器访问初次部署的Snake Demo应用，可以发现是一个“贪吃蛇”游戏： 
![](Images/3/visit-initial-snake.png) 
    
请记录Snake Demo应用的部署（Deployment）的名称，后续配置Jenkins自动部署时需要使用。  

### 1.3 配置自动部署    

为了实现应用更新之后的自动部署，我们需要修改Jenkinsfile Pipeline，增加自动部署环节。    

具体而言，需要修改GitLab代码库中的Jenkinsfile，在最后增加一个自动部署的Stage，如下所示：  

```
            stage('Deploy app to EKS') {
                //请按需修改Deployment名称和Snake镜像名称
                sh """kubectl set image deployment/snake-demo-snake-demo-cao7ea5d snake-demo-snake-demo-cao7ea5d=hub.easystack.io/3dc70621b8504c98/snake:v${BUILD_NUMBER}"""
            }
```
其中kubectl set image命令可以更新Deployment所使用的镜像版本，```deployment```参数需指定为Snake Demo应用的Deployment名称。   

### 1.4 配置自动触发构建    
为了实现GitLab中更新代码操作能够自动触发Jenkins Pipeline构建，我们需要在GitLab中配置Webhook。     
具体步骤如下：  
在GitLab的项目中选择【Settings】->【Integrations】，新建Webhook：  
![](Images/3/gitlab-add-webhook.png)   
其中URL需要填写准确，具体的格式可参考```http://<Jenkins user account>:<Jenkins user password>@<NodeIP>:<Jenkins Service NodePort>/project/<Jenkins project name>```  

>注：此处选择的触发方式（Trigger）为“Push events”，代表每次Push代码操作都会触发Webhook，您也可以选择其他触发（Trigger）方式，如“Merge Request events”等。  

添加成功后，点击“test”进行测试：   
![](Images/3/gitlab-webhook-test-1.png)  

如果返回“Hook executed successfully: HTTP 200 ”即表示Webhook配置成功：
![](Images/3/gitlab-webhook-test-2.png)  

后续每次往GitLab的“snake-demo”项目中Push代码后，将会自动触发Jenkins相对应的Pipeline进行构建，而无需手动启动Jenkins Pipeline。  


## 2. CI/CD演示    

在完成Snake Demo项目的CI/CD配置之后，我们可以演示CI/CD流程：  

```更新代码```->```自动构建镜像```->```上传镜像```->```自动部署```  

具体操作步骤参考如下：  

修改GitLab中Snake-demo项目的源代码下的css目录下的main-snake.css文件，可参考下图所示：    
![](Images/3/update-code.png)  
将```.snake-playing-field```的```background-color```参数从```#0000A8```（蓝色）修改为```#008000```（绿色）。 
修改代码并“Commit change”之后，会自动触发Jenkins Pipeline，执行CI/CD流程。   

在Jenkins Blue Ocean界面中查看Pipeline执行状态：  

![](Images/3/blueocean-pipeline-build.png)

![](Images/3/blueocean-pipeline-success.png)

等待自动部署完成后，刷新Snake Demo的Web页面，可以看到“贪吃蛇”游戏的背景颜色由原来的蓝色变为了绿色。 
![](Images/3/check-updated-app.png) 

同时，我们也可以在EKS界面查看Kubernetes Deployment所采用的镜像已经完成更新。   
![](Images/3/check-deployment-image-update.png)  

后续每次更新GitLab中的源代码，均会重复上述CI/CD流程。  

=================================================================================================================================

**Step 1: 在GitLab中创建示例项目。** 

在GitLab中创建一个示例项目“dubbo-demo”：  
![](Images/4/gitlab-create-project-1.png)  
![](Images/4/gitlab-create-project-2.png) 

**Step 2: 上传dubbo项目代码以及pom等文件到Gitlab项目中**

首先需要确认所使用的本地虚拟机环境中已经安装了Git，并完成Git Global Config配置。   
然后从GitHub上将示例项目的源代码克隆（Clone）到本地虚拟机中：  
```
[root@docker-ce ~]# git clone https://github.com/PabloZhong/dubbo-demo.git
```
可参考GitLab中界面提示，通过SSH方式进行源代码Push：  
```
[root@docker-ce ~]# cd dubbo/
[root@docker-ce dubbo]# rm -rf .git
[root@docker-ce dubbo]# git init  
[root@docker-ce dubbo]# git remote add origin ssh://git@gitlab.example.org:30022/easystack/dubbo-demo.git  
[root@docker-ce dubbo]# git add .  
[root@docker-ce dubbo]# git commit -m "Initial commit"  
[root@docker-ce dubbo]# git push -u origin master  
```

备注：需要在运行git命令的本机上修改hosts文件，加入IP gitlab.example.org这行

Push成功后即可在GitLab的“dubbo-demo”项目中看到已上传的源代码。  

**Step 6: 修改Dubbo源代码配置文件。** 

（1）dubbo/dubbo-demo/dubbo-demo-consumer/src/main/assembly/conf/dubbo.properties

dubbo.registry.address=zookeeper://172.16.2.245:2181

（2）dubbo/dubbo-demo/dubbo-demo-provider/src/main/assembly/conf/dubbo.properties
dubbo.registry.address=zookeeper://172.16.2.245:2181

其中dubbo.registry.address的地址是创建的zookeeper的地址。

3.2.2 构建dubbo镜像
使用以下pipeline script构建dubbo镜像：

```
podTemplate(name: 'jnlp', label: 'jnlp', namespace: 'default', cloud: 'kubernetes',
  containers: [
        containerTemplate(
            name: 'jnlp',
            image: 'hub.easystack.io/3dc70621b8504c98/jenkins-slave-maven:latest',
            command: '',
            args: '${computer.jnlpmac} ${computer.name}',
            privileged: true,
            alwaysPullImage: false,
            ttyEnabled: true, 
        ),
  ],
  volumes: [hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock'),
            hostPathVolume(hostPath: '/usr/bin/docker', mountPath: '/usr/bin/docker'),
            hostPathVolume(hostPath: '/usr/bin/docker-current', mountPath: '/usr/bin/docker-current'),
            hostPathVolume(hostPath: '/etc/sysconfig/docker', mountPath: '/etc/sysconfig/docker'),
            hostPathVolume(hostPath: '/usr/bin/kubectl', mountPath: '/usr/bin/kubectl')]
  ) {

  node('jnlp') {
    stage('devops for dubbo') {
        container('jnlp') {
            stage("clone dubbo code") {
                git 'https://github.com/PabloZhong/dubbo-1.git'
            }
            stage('compile') {
                
            
                echo 'Hello, Maven'
                sleep 10
                sh 'java -version'
                dir('/home/jenkins/workspace/dubbo-old/dubbo-demo')
                {
                
                sh '/opt/rh/rh-maven33/root/usr/bin/mvn clean install'
                }

            }
            
            stage('build docker image') {
                sh """
                    docker login -u 3dc70621b8504c98 -p Tcdf4f05247d79dd7 hub.easystack.io
                    
                    docker build -t hub.easystack.io/3dc70621b8504c98/dubbo-consumer:v11 /home/jenkins/workspace/dubbo-old/dubbo-demo/dubbo-demo-consumer
                    
                    docker push hub.easystack.io/3dc70621b8504c98/dubbo-consumer:v11
                                
                    docker build -t hub.easystack.io/3dc70621b8504c98/dubbo-provider:v11 /home/jenkins/workspace/dubbo-old/dubbo-demo/dubbo-demo-provider
                    
                    docker push hub.easystack.io/3dc70621b8504c98/dubbo-provider:v11

                    
                """
            }
            
        }
    }
 }
}
```
使用“image: 'hub.easystack.io/3dc70621b8504c98/jenkins-slave-maven:latest'”镜像作为jenkins的slave镜像， 
使用“git 'https://github.com/PabloZhong/dubbo-1.git'”拉取dubbo源码，使用“mvn clean install”编译dubbo代码，生成jar包。

使用“docker build -t hub.easystack.io/3dc70621b8504c98/dubbo-consumer:v11 /home/jenkins/workspace/dubbo-old/dubbo-demo/dubbo-demo-consumer”命令构建dubbo-consumer容器镜像，

使用“docker build -t hub.easystack.io/3dc70621b8504c98/dubbo-provider:v11 /home/jenkins/workspace/dubbo-old/dubbo-demo/dubbo-demo-provider”构建dubbo-provider容器镜像，
并通过“docker push”命令推送到镜像仓库中去。

构建成功后控制台结果输出如下：
![](Images/dubbo-control-1.png)
![](Images/dubbo-control-2.png)

blueocean输出如下：

![](Images/dubbo-bo.png)

其中dubbo-consumer的Dockefile文件如下：
```
FROM openjdk:8-jre
ADD target/dubbo-demo-consumer-2.5.7-assembly.tar.gz /dubbo
COPY start-docker.sh /dubbo/dubbo-demo-consumer-2.5.7/bin/

CMD ["sh","-c","/dubbo/dubbo-demo-consumer-2.5.7/bin/start-docker.sh"]

```
dubbo-provider的Dockfile文件如下：
```
FROM openjdk:8-jre
ADD target/dubbo-demo-provider-2.5.7-assembly.tar.gz /dubbo
COPY start-docker.sh /dubbo/dubbo-demo-provider-2.5.7/bin/

CMD ["sh","-c","/dubbo/dubbo-demo-provider-2.5.7/bin/start-docker.sh"]

```
在此要注意，需要修改dubbo源码路径下的dubbo-demo-consumer.xml和dubbo-demo-provider.xml文件中的zookeeper地址
![](Images/4/dubbo-consumer-xml.png)

![](Images/4/dubbo-provider-xml.png)

dubbo-consumer,dubbo-provider镜像成功构建并且推送到镜像仓库后，下一步要通过ECS和EKS部署使用dubbo。
3.3 在ECS+EKS环境里面将dubbo应用落地


在该场景里面采用ECS里面的大数据组件来实现zookeeper集群的快速部署，提供dubbo应用架构的服务注册中心，采用EKS来部署dubbo应用，dubbo应用分为两类，一类是提供服务的provider，另一类是消费服务的consumer，两类服务均采用容器部署的方式部署，部署步骤如下：

**Step 1 - 部署zookeeper集群作为dubbo应用中的注册中心**

1.创建zookeeper网络

![](Images/4/zk-network.png)

2.创建zookeeper集群

![](Images/4/create-zk.png)

**Step 2 - 部署dubbo应用**

1.创建应用依赖配置文件
```
dubbo.properties
[escore@ci-akyzklrim5-0-vsx3xunzxan2-kube-master-gko2lwdxza5r opt]$ cat dubbo.properties 
dubbo.container=log4j,spring
dubbo.application.name=demo-provider
dubbo.application.owner=
dubbo.registry.address=zookeeper://172.16.4.123:2181?backup=172.16.4.122:2181,172.16.4.124:2181
dubbo.monitor.protocol=registry
dubbo.protocol.name=dubbo
dubbo.protocol.port=20880
dubbo.service.loadbalance=roundrobin
dubbo.log4j.file=logs/dubbo-demo-provider.log
dubbo.log4j.level=WARN

```
2、创建应用依赖configmap

```
kubectl create configmap dubbo-config --from-file=dubbo.properties
```
3.在EKS环境里面部署dubbo-provider和dubbo-consumer应用

![](Images/4/provider-service.png)

![](Images/4/provider-service-2.png)

部署成功后，查看应用运行情况：

![](Images/4/check-service.png)

**场景 1 - 查看服务调用效果**

1.查看运行pod

![](Images/4/get-pod.png)

2.查看服务注册情况
登陆zookeeper集群：

![](Images/4/login-zk.png)

3.查看服务注册情况：

![](Images/4/check1.png)

从以上四步，我们可以看出provider服务在注册中心（zookeeper集群）注册成功，同时consumer调用成功，说明我们可以将dubbo场景下的微服务在ECS+EKS下落地。


