# CI/CD场景实践-Dubbo微服务应用 (Ready)  

## 1. 概述
Dubbo是目前流行的开源分布式微服务框架，其最大的特点是方便开发人员按照解耦的方式来开发应用，使用该框架可以使各个服务之间解耦合。从服务模型的角度，Dubbo采用的是一种非常简单的模型，由提供方提供服务，由消费方消费服务，基于这点可以抽象出服务提供方（Provider）和服务消费方（Consumer）两个角色。   

本文档主要介绍如何在已完成部署的CI/CD工具链基础上，实现基于Dubbo框架的微服务应用的CI/CD配置和演示。其中应用使用Java编程语言，采用Maven进行编译。CI/CD场景演示主要包括源代码上传、编译、镜像构建、推送镜像以及应用自动部署等。 

**应用架构描述**  
Dubbo微服务场景涉及的应用架构如下图所示：    
![](Images/4/dubbo-demo-architecture.png)  

其中Zookeeper集群作为服务注册中心。Dubbo应用分为两类，一类是提供服务的Provider，另一类是消费服务的Consumer。两类服务均采用Sidecar模式的Pod进行部署，也就是将应用程序与运行环境进行分离部署至同一个Pod的两个容器中。这样设计的好处是能够减小应用程序容器镜像的大小，同时可以复用运行环境容器镜像，因此在对应用变更时，可以只变更应用程序所使用的镜像，实现快速分发，提升CI/CD效率。   

**预计实操时间：约40-50分钟**   
**环境要求： ECS 4.0.2、EKS 4.0.2**   

## 2. CI/CD配置  

### 2.1 在GitLab中创建项目，并上传源代码  

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
（备注：需要提前在运行Git的虚拟机上修改/etc/hosts文件，加入集群节点NodeIP与域名的映射，如本示例加入172.16.6.28 gitlab.example.org）
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


### 2.2 创建Jenkins Pipeline    

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

**Step 2: 修改Jenkinsfile。**   
在GitLab的Dubbo-demo项目源代码中，包含有定义Jenkins Pipeline的Jenkinsfile，在第一次执行Pipeline之前，需要按照实际环境对Jenkinsfile做必要的修改。  
本示例源代码中的Jenkinsfile参考如下：
```
podTemplate(name: 'jnlp', label: 'jnlp', cloud: 'kubernetes',
  containers: [
        containerTemplate(
            name: 'jnlp',
            image: 'hub.easystack.io/3dc70621b8504c98/jenkins-slave-maven:v1',
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
    stage('CICD for Dubbo Demo') {
        container('jnlp') {
            stage("Clone source code of Dubbo Demo") {
                //请按需修改Git源代码库地址
                //如果是Private项目，参考示例如下（需使用GitLab Access Token）
                sh """
                    git clone http://oauth2:QK6YdGhk3muxTPzAQC1B@172.16.6.28:30080/easystack/dubbo-demo.git
                """
                //如果是Public项目，参考示例如下
                //git 'http://172.16.6.28:30080/easystack/dubbo-demo.git'
            }
            
            stage('Compile') {
                echo 'Hello, Maven!'
                sh 'java -version'
                dir('./dubbo-demo/dubbo-demo')
                {
                    sh '/opt/rh/rh-maven33/root/usr/bin/mvn clean install'
                }
            }
            
            stage('Build & push docker image') {
                //请按需修改镜像仓库的账号和密码，并注意docker build命令中Dockerfile所在路径
                sh """
                    docker login -u 3dc70621b8504c98 -p Tcdf4f05247d79dd7 hub.easystack.io
                    docker build -t hub.easystack.io/3dc70621b8504c98/dubbo-consumer:v${BUILD_NUMBER} ./dubbo-demo/dubbo-demo/dubbo-demo-consumer
                    docker push hub.easystack.io/3dc70621b8504c98/dubbo-consumer:v${BUILD_NUMBER}
                    docker build -t hub.easystack.io/3dc70621b8504c98/dubbo-provider:v${BUILD_NUMBER} ./dubbo-demo/dubbo-demo/dubbo-demo-provider
                    docker push hub.easystack.io/3dc70621b8504c98/dubbo-provider:v${BUILD_NUMBER}
                """
            //stage('Deploy app to EKS') {
                //更新包含应用程序jar包在内的dubbo-demo-jar容器所使用镜像
                //sh """kubectl set image deployment/dubbo-demo-provider dubbo-demo-jar=hub.easystack.io/3dc70621b8504c98/dubbo-provider:v${BUILD_NUMBER}"""
                //sh """kubectl set image deployment/dubbo-demo-consumer dubbo-demo-jar=hub.easystack.io/3dc70621b8504c98/dubbo-consumer:v${BUILD_NUMBER}"""
            //}           
        }
    }
 }
}
```
其中有以下几点需要说明，其中2）和4）请按照实际环境做适当的修改：  

1）```image: 'hub.easystack.io/3dc70621b8504c98/jenkins-slave-maven:v1'```指定之前Step 1中构建的Jenkins Slave镜像。   

2）```stage("Clone source code of Snake game")```将源代码从GitLab中拉取到Jenkins Slave Pod中，请按需修改。具体写法如下：    
   · 如果是Public类型的GitLab项目，直接通过HTTP方式Git clone源代码即可，无需使用用户名+密码或者Access Token；  
   · 如果是Private类型的GitLab项目，则需要使用```用户名+密码```或使用[文档2](./2_搭建CICD工具链.md)中生成的GitLab ```Access Token```，具体格式参考：  

```
    git clone http://<username>:<password>@<GitLab URL>/<username>/<project name>.git  
    或：  
    git clone http://oauth2:<access token>@<GitLab URL>/<username>/<project name>.git  
```  

3）```stage('Compile')```执行Maven编译，这个过程中需要拉取依赖包，因此耗时比较长（10-20分钟左右），编译成功后将生成应用包。  

4）下面的命令分别实现登录镜像仓库、构建Dubbo Demo镜像（分为Dubbo comsumer镜像和Dubbo Provider镜像），以及上传镜像，请按需修改：  
```
stage('Build & push docker image') {
    //请按需修改镜像仓库的账号和密码，并注意docker build命令中Dockerfile所在路径
    sh """
        docker login -u 3dc70621b8504c98 -p Tcdf4f05247d79dd7 hub.easystack.io
        docker build -t hub.easystack.io/3dc70621b8504c98/dubbo-consumer:v${BUILD_NUMBER} ./dubbo-demo/dubbo-demo/dubbo-demo-consumer
        docker push hub.easystack.io/3dc70621b8504c98/dubbo-consumer:v${BUILD_NUMBER}
        docker build -t hub.easystack.io/3dc70621b8504c98/dubbo-provider:v${BUILD_NUMBER} ./dubbo-demo/dubbo-demo/dubbo-demo-provider
        docker push hub.easystack.io/3dc70621b8504c98/dubbo-provider:v${BUILD_NUMBER}
    """
}
``` 
上面的docker build构建镜像步骤，需要使用Jenkins Slave从GitLab代码库中拉取的源代码中所包含的Dockerfile。    
其中生成```dubbo-consumer```镜像的Dockefile文件如下：
```
FROM docker.io/busybox:latest 
ADD target/dubbo-demo-consumer-2.5.7-assembly.tar.gz .
RUN mv dubbo-demo-consumer-2.5.7/ dubbo-demo-consumer/
COPY start-docker.sh dubbo-demo-consumer/bin/
CMD "tail" "-f" "/dev/null"
```

而生成```dubbo-provider```镜像的Dockfile文件如下：
```
FROM docker.io/busybox:latest 
ADD target/dubbo-demo-provider-2.5.7-assembly.tar.gz .
RUN mv dubbo-demo-provider-2.5.7/ dubbo-demo-provider/
COPY start-docker.sh dubbo-demo-provider/bin/
CMD "tail" "-f" "/dev/null"
```

**Step 3: 通过Jenkins Blue Ocean创建Jenkins Pipeline，并执行第一次Pipeline。**   
使用Jenkins Blue Ocean能够实现更丰富、更直观的Pipeline功能。  

在Jenkins主界面点击“Open Blue Ocean”进入Blue Ocean操作界面：   
![](Images/4/jenkins-blue-ocean.png)   

点击“创建流水线”：   
![](Images/4/jenkins-create-pipeline-1.png)   

填入GitLab代码仓库对应的项目地址：（注意：SSH的URL中需要将域名改成NodeIP）   
![](Images/4/jenkins-create-pipeline-2.png)  

Jenkins将自动生成SSH Key Pair，在创建Pipeline之前，需要将SSH公钥添加到GitLab中，添加路径为【GitLab】-【User Setting】-【SSH Keys】：  
![](Images/4/gitlab-ssh-key.png)  

回到Jenkins Blue Ocean界面，点击“创建Pipeline”:    
![](Images/4/jenkins-initial-pipeline-1.png)  

Jenkins首先将会自动拉取GitLab代码库中已经完成修改的Jenkinsfile，并按照Jenkinsfile执行第一次Pipeline： 
![](Images/4/jenkins-initial-pipeline-2.png)  

在Blue Ocean界面中可以查看Pipeline执行进度：   
![](Images/4/check-initial-pipeline.png)  

可以在EKS界面中看到正在执行Pipeline的Jenkins Slave Pod：   
![](Images/4/check-jenkins-slave-pod.png)  

执行完成第一次Pipleline后，可以在EKS的镜像仓库中查看第一次构建并上传的Dubbo comsumer和Dubbo provider镜像：  
![](Images/4/check-demo-image-1.png)  
![](Images/4/check-demo-image-2.png)  

注：按照上面所示的Jenkinsfile执行的Pipeline，第一次构建只会完成Dubbo Demo镜像构建并上传到EKS镜像仓库，下一步需要手动进行第一次应用部署。  

### 2.3 在EKS中完成Dubbo Demo应用的首次部署   
**Step 1: 部署Zookeeper集群作为Dubbo微服务应用注册中心。**   
我们可以使用ECS云平台中的Zookeeper集群服务，作为Dubbo微服务应用的注册中心。  
按照ECS界面提示完成创建Zookeeper集群前的准备工作：【创建私有网络】-【创建路由器】-【连接私有网络至路由器】-【设置路由器网关】。    
![](Images/4/create-zk-1.png)  

创建Zookeeper集群服务：  
![](Images/4/create-zk-2.png)  

等待Zookeeper集群服务正常运行：  
![](Images/4/create-zk-3.png)  

记录Zookeeper实例的管理网络IP：  
![](Images/4/create-zk-4.png)  

**Step 2: 创建应用依赖配置文件。**  
本示例中Dubbo-demo应用所需要的配置文件```dubbo.properties```如下：  
```
dubbo.container=log4j,spring
dubbo.application.name=demo-provider
dubbo.application.owner=
dubbo.registry.address=zookeeper://172.16.7.44:2181?backup=172.16.7.45:2181,172.16.7.46:2181
dubbo.monitor.protocol=registry
dubbo.protocol.name=dubbo
dubbo.protocol.port=20880
dubbo.service.loadbalance=roundrobin
dubbo.log4j.file=logs/dubbo-demo-provider.log
dubbo.log4j.level=WARN
``` 
>注：其中dubbo.registry.address为Zookeeper地址。另外，该配置文件在源代码中的路径为dubbo-demo/dubbo-demo-consumer/src/main/assembly/conf/dubbo.properties和dubbo-demo/dubbo-demo-provider/src/main/assembly/conf/dubbo.properties。
使用SSH私钥登录EKS集群Master节点，创建```dubbo.properties```配置文件：  
![](Images/4/create-dubbo-config.png)  

使用```dubbo.properties```配置文件创建Kubernetes Configmap，命名为“dubbo-config”：  
```
[root@ci-ebapjqyquc-0-us73jutasdon-kube-master-vpstuakkzsnl escore]# kubectl create configmap dubbo-config --from-file=dubbo.properties
```  

查看已创建的Configmap：  
![](Images/4/check-dubbo-configmap.png)  

下一步在部署应用时，可以将配置文件通过Configmap的方式注入至应用容器中，达到应用和配置分离的目的，方便应用镜像在不同环境（如开发、测试、准生产、生产等）中的共享和迁移。  
 
**Step 3: 在EKS中部署Dubbo-demo-provider应用。**  
采用Sidecar模式部署Dubbo-demo-provider应用，需要直接通过Yaml文件部署。  
在ESK界面点击【创建应用】：  
![](Images/4/create-dubbo-demo-app-1.png)  

选择通过“编排模板”创建应用：  
![](Images/4/create-dubbo-demo-app-2.png)  

接下来我们需要在编辑框中填入Dubbo-demo-provider应用的Deployment编排文件，如下：（注：其中```dubbo-provider```镜像名称请按需修改）   
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: dubbo-demo-provider
  namespace: default
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: dubbo-demo-provider
    spec:
      containers:
      - image: hub.easystack.io/3dc70621b8504c98/dubbo-provider:v1
        resources:
          limits:
            cpu: 400m
            memory: 400Mi
          requests:
            cpu: 200m
            memory: 200Mi
        imagePullPolicy: Always
        name: dubbo-demo-jar
        lifecycle:
          postStart:
            exec:
              command:
                - "mv"
                - "/dubbo-demo-provider"
                - "/app"
        volumeMounts:
        - mountPath: /app
          name: app-volume    
      - image: openjdk:8-jre
        resources:
          limits:
            cpu: 400m
            memory: 400Mi
          requests:
            cpu: 200m
            memory: 200Mi
        imagePullPolicy: IfNotPresent
        name: dubbo-demo-jdk
        command: ["sh","-c","/app/dubbo-demo-provider/bin/start-docker.sh"]
        volumeMounts:
        - mountPath: /app
          name: app-volume
        - mountPath: /app/dubbo-demo-provider/conf
          name: config-volume
        ports:
        - containerPort: 8080        
      volumes:
      - name: app-volume
        emptyDir: {}
      - name: config-volume
        configMap:
          name: dubbo-config
```
其中需要通过Configmap挂载之前创建的```dubbo-config```配置文件。  

点击“创建”开始部署应用：  
![](Images/4/create-dubbo-demo-app-3.png)  

可以查看处于“运行中”状态的Dubbo-demo-provider应用：  
![](Images/4/create-dubbo-demo-app-4.png)  

**Step 4: 在EKS中部署Dubbo-demo-consumer应用。**  

创建部署与上同，其中使用的Deployment编排模板如下： （注：其中```dubbo-consumer```镜像名称请按需修改） 
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: dubbo-demo-consumer
  namespace: default
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: dubbo-demo-consumer
    spec:
      containers:
      - image: hub.easystack.io/3dc70621b8504c98/dubbo-consumer:v1
        resources:
          limits:
            cpu: 400m
            memory: 400Mi
          requests:
            cpu: 200m
            memory: 200Mi
        imagePullPolicy: Always
        name: dubbo-demo-jar
        lifecycle:
          postStart:
            exec:
              command:
                - "mv"
                - "/dubbo-demo-consumer"
                - "/app"
        volumeMounts:
        - mountPath: /app
          name: app-volume    
      - image: openjdk:8-jre
        resources:
          limits:
            cpu: 400m
            memory: 400Mi
          requests:
            cpu: 200m
            memory: 200Mi
        imagePullPolicy: IfNotPresent
        name: dubbo-demo-jdk
        command: ["sh","-c","/app/dubbo-demo-consumer/bin/start-docker.sh"]
        volumeMounts:
        - mountPath: /app
          name: app-volume
        - mountPath: /app/dubbo-demo-consumer/conf
          name: config-volume
        ports:
        - containerPort: 8080        
      volumes:
      - name: app-volume
        emptyDir: {}
      - name: config-volume
        configMap:
          name: dubbo-config
```
可以查看处于“运行中”状态的Dubbo-demo-consumer应用：  
![](Images/4/create-dubbo-demo-app-5.png)  

**Step 5: 查看服务注册情况。**  
首先通过创建Zookeeper集群服务时使用的SSH密钥，登录Zookeeper后台：  
```
[root@docker-ce ~]# ssh -i ~/.ssh/id_rsa_zk ubuntu@172.16.7.44
``` 
执行以下命令行：  
```
ubuntu@dubbo-zk-dubbo-zkwrk-1:~$ cd /opt/zookeeper/zookeeper/bin/
ubuntu@dubbo-zk-dubbo-zkwrk-1:/opt/zookeeper/zookeeper/bin$ ./zkCli.sh
```
通过Zookeeper命令行查看服务注册情况：  
![](Images/4/check-dubbo-service-registration.png)  

**Step 6: 查看Dubbo-demo微服务应用运行情况。**  
可以通过EKS界面查看Dubbo-demo-provider Pod中的```dubbo-demo-jdk```容器输出日志：  
![](Images/4/check-provider-logs.png)  
可以看到容器输出“Hello world”。  

可以通过EKS界面查看Dubbo-demo-consumer Pod中的```dubbo-demo-jdk```容器输出日志： 
![](Images/4/check-consumer-logs.png)  
可以看到容器输出“Hello world”。  

当然也可以通过SSH密钥登录EKS的Master节点，使用后台命令行查看容器输出日志，如：  
```
[root@ci-ebapjqyquc-0-us73jutasdon-kube-master-vpstuakkzsnl escore]# kubectl get pod  
[root@ci-ebapjqyquc-0-us73jutasdon-kube-master-vpstuakkzsnl escore]# kubectl logs -f dubbo-demo-consumer-3861726430-j0kkw  -c dubbo-demo-jdk  
``` 
查看容器日志输出：  
![](Images/4/check-initial-app.png) 

### 2.4 配置自动部署    

为了实现应用更新之后的自动部署，我们需要修改Jenkinsfile Pipeline，增加自动部署环节。    

具体而言，需要修改GitLab代码库中的Jenkinsfile，在最后增加一个自动部署的Stage，如下所示：  

```
      stage('Deploy app to EKS') {
          //更新包含应用程序jar包在内的dubbo-demo-jar容器所使用镜像 
          sh """kubectl set image deployment/dubbo-demo-provider dubbo-demo-jar=hub.easystack.io/3dc70621b8504c98/dubbo-provider:v${BUILD_NUMBER}"""
          sh """kubectl set image deployment/dubbo-demo-consumer dubbo-demo-jar=hub.easystack.io/3dc70621b8504c98/dubbo-consumer:v${BUILD_NUMBER}"""
      }
```
其中kubectl set image命令可以更新Deployment所使用的镜像版本，```deployment```参数需指定为Dubbo-demo应用的Deployment名称。
>注：采用Sidecar类型的Pod进行dubbo-demo-provider和dubbo-demo-consumer部署后，后续更新版本的自动部署，只需要更新Pod中的dubbo-demo-jar这一个容器即可，而Dubbo基础环境容器（即dubbo-demo-jdk）无需更新。通过这种方式可以达到提升更新效率的目的。     

### 2.5 配置自动触发构建    
为了实现GitLab中更新代码操作能够自动触发Jenkins Pipeline构建，我们需要在GitLab中配置Webhook。     
具体步骤如下：  
在GitLab的项目中选择【Settings】->【Integrations】，新建Webhook：  
![](Images/4/gitlab-add-webhook.png)   
其中URL需要填写准确，具体的格式可参考```http://<Jenkins user account>:<Jenkins user password>@<NodeIP>:<Jenkins Service NodePort>/project/<Jenkins project name>```  

>注：此处选择的触发方式（Trigger）为“Push events”，代表每次Push代码操作都会触发Webhook，您也可以选择其他触发（Trigger）方式，如“Merge Request events”等。  

添加成功后，点击“test”进行测试：   
![](Images/4/gitlab-webhook-test-1.png)  

如果返回“Hook executed successfully: HTTP 200 ”即表示Webhook配置成功：
![](Images/4/gitlab-webhook-test-2.png)  

后续每次往GitLab的“dubbo-demo”项目中Push代码后，将会自动触发Jenkins相对应的Pipeline进行构建，而无需手动启动Jenkins Pipeline。  


## 3. CI/CD演示    

在完成Dubbo Demo项目的首次部署和CI/CD配置之后，我们可以演示CI/CD流程：  

```更新代码```->```自动构建镜像```->```上传镜像```->```自动部署```  

具体操作步骤参考如下：  

修改GitLab中Dubbo-demo-consumer源代码下的```dubbo-demo/dubbo-demo-provider/src/main/java/com/alibaba/dubbo/demo/consumer/Consumer.java```文件，可参考下图所示：    
![](Images/4/update-code.png)  
将```String hello = demoService.sayHello("world");```修改为```String hello = demoService.sayHello("world v2");```。 
修改代码并“Commit change”之后，会自动触发Jenkins Pipeline，执行CI/CD流程。   

在Jenkins Blue Ocean界面中查看Pipeline执行状态：  

![](Images/4/blueocean-pipeline-build.png)

CI/CD执行完毕：
![](Images/4/blueocean-pipeline-success.png)

等待自动部署完成后，使用SSH私钥登录EKS集群Master节点，输入命令查看Dubbo-demo-consumer Pod中的```dubbo-demo-jdk```容器输出日志： 
```
[root@ci-ebapjqyquc-0-us73jutasdon-kube-master-vpstuakkzsnl escore]# kubectl get pod  
[root@ci-ebapjqyquc-0-us73jutasdon-kube-master-vpstuakkzsnl escore]# kubectl logs -f dubbo-demo-consumer-1847343400-v8vq9  -c dubbo-demo-jdk  
``` 
可以看到输出已经变为“Hello world v2”：  
![](Images/4/check-updated-app.png) 

同时，我们也可以在EKS界面查看Kubernetes Deployment所采用的镜像已经完成更新。   
![](Images/4/check-deployment-image-update.png)  

后续每次更新GitLab中的源代码，均会重复上述CI/CD流程。  
