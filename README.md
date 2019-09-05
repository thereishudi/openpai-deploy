## 背景

最近决定使用微软开源的Microsoft/pai来作为我们平常深度算法模型的训练平台。 要部署pai（本文时的版本是v0.14.0），按照官方文档的说法，需要以下软硬件资源：


1. Ubuntu 16.04；
2. 机器需要联网并且有静态IP；
3. 机器可以访问Docker registry service (e.g., Docker hub)；
4. 机器的SSH service需要运行，username需要有sudo权限；
5. 如果有多台机器的话，使用同样的username/password，并且username都需要有sudo权限；
6. （非单机版）开启NTP服务；
7. 没有安装Docker，或者安装的Docker的api version >= 1.26；没有安装nvidia驱动。
8. 机器需要有至少40G内存（单机）或者16G内存（集群方式部署）；内存主要提供给K8s和PAI服务。

由于我手头只有一台32G内存的机器，所以是按照官方文档中的单机版模式来安装的。

## OpenPAI部署简介

OpenPAI基于Kubernetes，因此OpenPAI的安装脚本会自动安装Kubernetes。具体来说，就是要安装组件kube-scheduler、kube-controller-manager、kube-apiserver、kubernetes-dashboard、kube-proxy、etcd。注意，这些组件本身也是Docker化的。

在安装完K8s后，OpenPAI就要部署自己的服务了，服务的组件是frameworklauncher、hadoop-name-node、hadoop-jobhistory-service、hadoop-resource-manager、pylon、rest-server、webportal、zookeeper、end-to-end-test-deployment、grafana、prometheus、watchdog、drivers、node-exporter、hadoop-data-node、hadoop-node-manager。注意，这些组件本身也是Docker化的。

## 安装依赖
``` 

apt-get -y update 

apt-get -y install \
      nano \
      vim \
      joe \
      wget \
      curl \
      jq \
      gawk \
      psmisc \
      python \
      python-yaml \
      python-jinja2 \
      python-paramiko \
      python-urllib3 \
      python-tz \
      python-nose \
      python-prettytable \
      python-netifaces \
      python-dev \
      python-pip \
      python-mysqldb \
      openjdk-8-jre \
      openjdk-8-jdk \
      openssh-server \
      openssh-client \
      git \
      bash-completion \
      inotify-tools \
      rsync \
      coreutils \
      net-tools

pip install python-etcd docker kubernetes GitPython
```


本来不需要自己安装的，但是因为墙的存在，我们无法直接安装K8s的一些命令。以及要配置一些软件源，因此在这里借助国内镜像先行安装：

```

sudo apt-get update
sudo apt-get install -y apt-transport-https
#使用阿里云镜像
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt-add-repository "deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main"

#安装 nvidia-docker
curl -s -L https://nvidia.github.io/nvidia-container-runtime/gpgkey | sudo apt-key add -
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-container-runtime/$distribution/nvidia-container-runtime.list | sudo tee /etc/apt/sources.list.d/nvidia-container-runtime.list

#安装k8s的工具
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```


----------------------------------------------------
## 克隆openpai项目
`
$ git clone https://github.com/Microsoft/pai.git
`

## 3.  配置

修改deployment/quick-start/quick-start-example.yaml文件，配置IP和ssh用户密码：

```

# Special case: If this list contains only one machine, then this machine will be the master
 # and the worker at the same time.
 machines:
-  - <ip-of-master>
-  - <ip-of-worker1>
-  - <ip-of-worder2>
+  - 10.20.61.88
 
 # (Required) Log-in info of all machines. System administrator should guarantee
 # that the username/password pair or username/key-filename is valid and has sudo privilege.
-ssh-username: <username>
-ssh-password: <password>
+ssh-username: pai
+ssh-password: pai-password
```


然后使用下面的命令生成真正的配置文件：~/pai-config

```

python paictl.py config generate -i deployment/quick-start/quick-start-example.yaml -o ~/pai-config -f

```


## 4. 更改一些源文件（因为机器内存不够!!!）

主要改两类，一是k8s的yaml，另外一个是shell启动脚本。


更改以下的k8s yaml模板文件：
```

deployment/k8sPaiLibrary/template/apiserver.yaml.template
deployment/k8sPaiLibrary/template/controller-manager.yaml.template
deployment/k8sPaiLibrary/template/dashboard-deployment.yaml.template
deployment/k8sPaiLibrary/template/etcd.yaml.template
deployment/k8sPaiLibrary/template/scheduler.yaml.template
src/cleaner/deploy/cleaner.yaml.template
src/hadoop-jobhistory/deploy/hadoop-jobhistory.yaml.template
src/hadoop-name-node/deploy/hadoop-name-node.yaml.template
src/hadoop-node-manager/deploy/hadoop-node-manager.yaml.template
src/hadoop-resource-manager/deploy/hadoop-resource-manager.yaml.template
src/pylon/deploy/pylon.yaml.template
src/yarn-frameworklauncher/deploy/yarn-frameworklauncher.yaml.template
src/zookeeper/deploy/zookeeper.yaml.template
```


将其中的memory request适当减少(再次强调，**这一步只是因为机器内存不够**），以其中一个文件举例如下：

```

resources:
       requests:
-        memory: "1Gi"
+        memory: "512Mi"
         cpu: "1000m"
```


另外，更改shell脚本src/hadoop-node-manager/deploy/hadoop-node-manager-configuration/nodemanager-generate-script.sh，将其中的保留内存从40G改为20G：

```
let mem_reserved=16*1024
 else
     echo "Node role is 'Master & Worker'. Reserve 40G for os and k8s."
-    let mem_reserved=40*1024
+    let mem_reserved=20*1024
 fi
 let mem_total=(mem_total/1024/1024*1024)-mem_reserved
 sed  -i "s/{mem_total}/${mem_total}/g" $HADOOP_CONF_DIR/yarn-site.xml
```

## 5. 安装并启动K8s
`
sudo python paictl.py cluster k8s-bootup -p ~/pai-config`


这一步会自动做以下事情：


1. 安装kubectl(前面因为墙的缘故已经提前做了)；

2. 基于cluster-configuration.yaml, kubernetes-configuration.yaml和
   k8s-role-definition.yaml生成K8s相关的配置文件；

3. 使用kubectl启动Kubernetes；


安装成功后可以通过浏览器访问K8s的dashboard来验证：http://<master>:9090。

**但是这一步很奇怪的是一直连接不上本机的端口，请求一直被拒绝。。。。。
但是还是按照步骤继续部署下去**




如果想回退这一步的话，可以使用下面的命令来清理掉刚才的安装：
`sudo python paictl.py cluster k8s-clean -p ~/pai-config/`
 
 
 ## 6. 更新cluster配置信息
 
sudo python paictl.py config push -p ~/pai-config/

## 7. 安装并启动OpenPAI的服务

`sudo python paictl.py service start`


安装并启动成功后，可以通过访问dashboard的这个页面也加以验证：http://<master>:9090/#!/pod?namespace=default



如果想回退这一步的话，可以使用下面的命令清理掉刚才的安装：

`
sudo python paictl.py service stop
`
  ## 8. 安装完成
  

