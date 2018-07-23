# CI/CD场景实践-JavaScript应用 （待修改） 

## 1. 在GitLab中创建项目，并上传源代码  

**Step 1: 在GitLab中创建示例项目。**  
我们在GitLab中创建一个示例项目（Create a project），填入项目名称如“snake-demo”：  
![](Images/3/gitlab-create-project-1.png)  
![](Images/3/gitlab-create-project-2.png)  
注：目前只支持Public类型的GitLab项目，还需要研究如何支持Private项目。  

**Step 2: 上传源代码至GitLab示例项目。**  

首先需要确认所使用的本地虚拟机环境（可使用之前上传Docker镜像的环境）中已经安装了Git，并完成Git Global Config配置。   
然后从GitHub上将示例项目的源代码克隆（Clone）到本地虚拟机中：  
```
[root@docker-ce ~]# git clone https://github.com/PabloZhong/Snake.git
```

可参考GitLab中界面提示，通过SSH方式将源代码Push到GitLab中：  
（备注：需要提前在运行Git的虚拟机上修改/etc/hosts文件，加入集群节点NodeIP与域名的映射，如本示例加入172.16.6.48 gitlab.example.org）
```
[root@docker-ce ~]# cd Snake/
[root@docker-ce Snake]# git init
[root@docker-ce Snake]# git remote rename origin old-origin 
[root@docker-ce Snake]# git remote add origin ssh://git@gitlab.example.org:30022/easystack/snake-demo.git  
[root@docker-ce Snake]# git push -u origin --all   
```
其中GitLab项目地址参考：  
![](Images/3/gitlab-ssh-url.png)  

Push成功后即可在GitLab的“snake-demo”项目中看到已上传的源代码：  
![](Images/3/gitlab-check-source-code.png)  

## 2. 创建Jenkins Pipeline，并部署Snake应用    

**Step 1: 制作用于编译Snake源码的Jenkins Slave镜像。**  
为了完成使用jenkins slave来进行CI工作，需要自己制作一个jenkins slave镜像，并上传到EKS的镜像仓库中，自己制作jenkins slave镜像步骤如下：

1） 编写Dockefile如下：（参考https://github.com/PabloZhong/jenkins-1/tree/master/slave-base） 
```
FROM openshift/origin

MAINTAINER Ben Parees <bparees@redhat.com>

ENV HOME=/home/jenkins

USER root
# Install headless Java
RUN yum install -y centos-release-scl-rh && \
    x86_EXTRA_RPMS=$(if [ "$(uname -m)" == "x86_64" ]; then echo -n java-1.8.0-openjdk-headless.i686 ; fi) && \
    INSTALL_PKGS="bc gettext git java-1.8.0-openjdk-headless lsof rsync tar unzip which zip bzip2" && \
    yum install -y --setopt=tsflags=nodocs install $INSTALL_PKGS $x86_EXTRA_RPMS && \
    # have temporarily removed the validation for java to work around known problem fixed in fedora; jupierce and gmontero are working with
    # the requisit folks to get that addressed ... will switch back to rpm -V $INSTALL_PKGS when that occurs
    rpm -V bc gettext git lsof rsync tar unzip which zip bzip2  && \
    yum clean all && \
    mkdir -p /home/jenkins && \
    chown -R 1001:0 /home/jenkins && \
    chmod -R g+w /home/jenkins && \
    chmod 664 /etc/passwd && \
    chmod -R 775 /etc/alternatives && \
    chmod -R 775 /var/lib/alternatives && \
    chmod -R 775 /usr/lib/jvm && \
    chmod 775 /usr/bin && \
    chmod 775 /usr/lib/jvm-exports && \
    chmod 775 /usr/share/man/man1 && \
    chmod 775 /var/lib/origin && \    
    unlink /usr/bin/java && \
    unlink /usr/bin/jjs && \
    unlink /usr/bin/keytool && \
    unlink /usr/bin/orbd && \
    unlink /usr/bin/pack200 && \
    unlink /usr/bin/policytool && \
    unlink /usr/bin/rmid && \
    unlink /usr/bin/rmiregistry && \
    unlink /usr/bin/servertool && \
    unlink /usr/bin/tnameserv && \
    unlink /usr/bin/unpack200 && \
    unlink /usr/lib/jvm-exports/jre && \
    unlink /usr/share/man/man1/java.1.gz && \
    unlink /usr/share/man/man1/jjs.1.gz && \
    unlink /usr/share/man/man1/keytool.1.gz && \
    unlink /usr/share/man/man1/orbd.1.gz && \
    unlink /usr/share/man/man1/pack200.1.gz && \
    unlink /usr/share/man/man1/policytool.1.gz && \
    unlink /usr/share/man/man1/rmid.1.gz && \
    unlink /usr/share/man/man1/rmiregistry.1.gz && \
    unlink /usr/share/man/man1/servertool.1.gz && \
    unlink /usr/share/man/man1/tnameserv.1.gz && \
    unlink /usr/share/man/man1/unpack200.1.gz

# Copy the entrypoint
ADD contrib/bin/* /usr/local/bin/

# Run the Jenkins JNLP client
ENTRYPOINT ["/usr/local/bin/run-jnlp-client"]
```

2）构建Jenkins Slave镜像  
在Dockerfile所在的路径下执行以下命令进行镜像构建： 
```
[root@docker-ce jenkins-slave]# docker build -t jenkins-slave:v1 .
```

3）上传Jenkins Slave镜像  
Jenkins Slave镜像制作完成后，使用docker push命令将Jenkins Slave镜像上传到EKS的镜像仓库中。  
```
[root@docker-ce jenkins-slave]# docker push 172.16.0.176/3dc70621b8504c98/jenkins-slave:v1
```
上传成功后可查看已上传的镜像： 
![](Images/3/jenkins-slave-docker.png) 
后续步骤中会使用上面的镜像执行Jenkins Pipeline。  

**Step 2: 通过Blue Ocean创建Jenkins Pipeline。**  

进入BlueOcean：  
![](Images/3/jenkins-blue-ocean.png)   

点击“创建流水线”：   
![](Images/3/jenkins-create-pipeline-1.png)   
填入GitLab代码仓库对应的项目地址：（注：Blue Ocean默认需要使用SSH方式）   
![](Images/3/jenkins-create-pipeline-2.png)  
需要将Jenkins自动生成的SSH公钥添加到GitLab中：  
![](Images/3/gitlab-ssh-key.png)  

回到Jenkins Blue Ocean界面，点击“创建Pipeline”之后，将会自动搜索代码库中的Jenkinsfile，并按照Jenkins执行第一次Pipeline：  

![](Images/3/图缺.png)  

其中Jenkinsfile如下：
```
podTemplate(name: 'jnlp', label: 'jnlp', namesapce: 'default', cloud: 'kubernetes',
  containers: [
        containerTemplate(
            name: 'jnlp',
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
    stage('devops for snake game') {
        container('jnlp') {
            stage("clone snake code") {
                git 'http://172.16.6.30:30080/easystack/snake-demo1.git'
            }
            
            stage('unit test') {
                sh 'echo "unit test command"'
            }
            
            stage('build docker image') {
                sh """
                    docker login -u 3dc70621b8504c98 -p Tcdf4f05247d79dd7 hub.easystack.io
                    docker build -t hub.easystack.io/3dc70621b8504c98/snake:${BUILD_NUMBER} .
                    docker push hub.easystack.io/3dc70621b8504c98/snake:${BUILD_NUMBER}
                """
            }
            
            //stage('Deploy to EKS') 
            //   sh """kubectl set image deployment/snake-snake-e8fluud7 snake-snake-e8fluud7=hub.easystack.io/3dc70621b8504c98/snake:${BUILD_NUMBER}"""
            //}
        }
    }
 }
}
```

其中“  image: 'hub.easystack.io/3dc70621b8504c98/jenkins-slave:v1'”指明我们前面构建的jenkins slave镜像。
“git 'https://github.com/luluwangwang1989/Snake.git'”将snake源码从github上拉取下来（请按需修改源代码项目地址）。
使用
```
 stage('build docker image') {
                sh """
                    docker login -u 3dc70621b8504c98 -p Tcdf4f05247d79dd7 hub.easystack.io
                    docker build -t hub.easystack.io/3dc70621b8504c98/snake:${BUILD_NUMBER} .
                    docker push hub.easystack.io/3dc70621b8504c98/snake:${BUILD_NUMBER}
                """
            }
```
上述几步将snake build成docker 镜像，并push到EKS的镜像仓库中。
![](Images/3/check-snake-image.png) 

注：按照上面所示的Jenkinsfile执行的Pipeline，第一次构建只会完成Snake Demo镜像构建并上传到EKS镜像仓库，下一步需要手动进行第一次应用部署。  

在EKS中进行Snake Demo应用的第一次部署，使用EKS将构建成功的Snake镜像进行部署： 
（部署图缺） 
![](Images/3/check-snake-service.png)

访问初次部署的Snake Demo应用，可以发现是一个“贪吃蛇”游戏：：
![](Images/3/visit-snake.png)

    
请记录部署（Deployment）的名称，后续配置Jenkins自动部署时需要用到。  

**Step 3: 配置自动部署。**   
修改Jenkinsfile源代码，加上自动部署。  

其中snake目录结构以及jenkinsfile如下：
```
[root@docker-ce ~]# cd Snake
[root@docker-ce Snake]# ls
css  Dockerfile  index.html  Jenkinsfile  js  LICENSE  README.md
[root@docker-ce Snake]# cat Jenkinsfile
podTemplate(name: 'jnlp', label: 'jnlp', namesapce: 'default', cloud: 'kubernetes',
  containers: [
        containerTemplate(
            name: 'jnlp',
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
    stage('devops for snake game') {
        container('jnlp') {
            stage("clone snake code") {
                 git 'http://172.16.6.30:30080/easystack/snack-demo.git'
            }
            
            stage('unit test') {
                sh 'echo "unit test command"'
            }
            
            stage('build docker image') {
                sh """
                    docker login -u 3dc70621b8504c98 -p Tcdf4f05247d79dd7 hub.easystack.io
                    docker build -t hub.easystack.io/3dc70621b8504c98/snake:${BUILD_NUMBER} .
                    docker push hub.easystack.io/3dc70621b8504c98/snake:${BUILD_NUMBER}
                """
            }
            
            stage('deploy to k8s') {
                
                sh """kubectl set image deployment/snake snake=hub.easystack.io/captain/snake:${BUILD_NUMBER}"""
            }
        }
    }
 }
}
```

**Step 4: 配置Webhook实现自动触发构建。**      

在GitLab的项目中选择【Settings】->[Integrations]，构建Webhook  
![](Images/3/gitlab-integration-1.png)
![](Images/3/gitlab-integration-2.png)
添加成功后，点击此webhook后面的test进行测试   
在GitLab中测试连通性：  
![](Images/3/gitlab-webhook-test-1.png)  

如果返回Hook successfully executed,表示配置成功。
![](Images/3/gitlab-webhook-test-2.png)  

这样，下次push代码后，就会自动触发Jenkins上相对应的Pipeline进行构建，无需手动启动Jenkins Pipeline。  


## 3. CI/CD演示    

在gitlab中修改snake中的文件均会触发Jenkins自动构建snake项目：

修改snake代码中食物的颜色，并自动部署新的snake镜像：通过修改snake代码下的css文件中的 main-snake.css中的
![](Images/3/foodbody.png)
来修改食物的颜色，修改成功后，会自动触发Jenkins Pipeline，执行“代码克隆”->“镜像构建”->“镜像上传”->“自动部署”一整套流程。  

在Jenkins Blue Ocean界面中查看Pipeline执行状态：  

![](Images/3/blueoceanbuild.png)

![](Images/3/pipeline-success.png)

部署完成后，效果如下：
![](Images/3/changefoodcolor.png)
可以看到，食物的颜色由原来的大红色变为了黄色。至此，完成了修改snake源码自动构建snake镜像，并且自动部署snake服务的CI/CD流程。  
