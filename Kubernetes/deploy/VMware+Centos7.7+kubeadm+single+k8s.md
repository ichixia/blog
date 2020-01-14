# VMware+Centos7.7+kubeadm部署单机版k8s
#### 环境
1. 虚拟机 VMware Workstation 12 Pro(win 7)   
2. 系统 CentOS Linux release 7.7.1908 (Core)， 最好是最新的版本，在7.2版本的时候安装了很久都失败了    
4. 系统配置，4G 2CPU ，最好大于2C2G，不然可能会有坑
3. Docker版本 19.03.5
4. kubelet版本 v1.17.0  
5. kubeadm版本 v1.17.0  

#### 安装步骤简述  
1. 设置yum源，为安装docker、kubelet、kubectl、kubelet、kubeadm准备  
2. 安装docker  
3. kubeadm安装k8s  
    * kubelet、kubectl、kubernetes-cni
    * k8s的核心组件通过容器部署(etcd、kube-apiserver、kube-controller-manager、kube-scheduler)
    * 插件 CoreDNS、kube-proxy
4. 安装dashboard  

#### 安装过程中一些坑  
在安装过程中一些系统设置、镜像问题导致安装过程不能顺利进行的坑，现在这里汇总一下  
1. 系统  
这里用了centos 7.7.1908的最小化镜像安装的系统，曾经用过centos 7.2安装过，但是不能成功，网上说最好用最新的系统吧  

2. 系统配置，这个前面也讲过了，配置最好高于 2C2G吧

3. 这边docker和k8s都用了最新的版本，第一次在装的时候看着别人的教程，教程里面的版本不是最新的了，安装了之后有时候会出现一些bug，当然这些bug在新版本就处理好了  

4. 没有科学上网的环境安装docker和k8s最好配置国内的镜像源  

5. 通过容器部署k8s的核心组件以及插件，需要拉取镜像，很大一部分都是从k8s.gcr.io中拉取，这个网络是被墙的，镜像会拉取失败，解决办法大概有三种，这里我选择第一种  
    * 大部分的镜像可以中阿里云的镜像仓库中拉取，然后把镜像名称通过`docker tag`打成需要的镜像名称
    ```
    # 我想要拉取 k8s.gcr.io/kube-apiserver:v1.17.0 镜像
    # 可以先拉取 registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.17.0
    # 再通过docker tag 打多一个想要的镜像的名称
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.17.0
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.17.0 k8s.gcr.io/kube-apiserver:v1.17.0
    ```
    * 在用kubeadm进行安装k8s组件的时候，会有个配置文件修改从什么地方拉取什么镜像，直接把配置文件里面相关的镜像改成阿里云上面的镜像名称就可以了  
    * 让系统或者docker能够翻墙吧，这样就能拉取墙外的镜像了，docker有个配置是关于网络代理的
6. 有些系统配置可以先设置，这些在安装过程中都会需要的，如下：  
    ```
    # 关闭防火墙  
    systemctl stop firewalld & systemctl disable firewalld  
    
    # 关闭swap
    swapoff -a  
    # 修改/etc/fstab文件，注释掉包含“swap”的那一行
    # 重启系统；
    # 通过free -m命令查看swap是否关闭，如果关闭，则输出如下
    [root@vm-node1 ~]# free -m
                  total        used        free      shared  buff/cache   available
    Mem:           3770        1062         758          12        1949        2445
    Swap:             0           0           0
    
    # 关闭Selinux
    setenforce 0 
    # 修改/etc/sysconfig/selinux文件 
    SELINUX=disabled
    
    # 创建/etc/sysctl.d/k8s.conf，内容如下
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    # 创建完成后，执行
    sysctl --system
    ```
7. 安装完成之后最好用`systemctl enable kubelet`把kubelet设置为开机自启，这样在重启之后就不会全部的核心组件容器都在stop状态中

8. **当你在第一次装k8s，看到上面这些配置汇总不知道如何**，没有关系在后面进行安装的时候仔细看看安装步骤就好  

#### 详细安装步骤 
1. 安装docker k8s核心组件   
* 参考这个 [链接](https://www.jianshu.com/p/70efa1b853f5)   `https://www.jianshu.com/p/70efa1b853f5`   每个步骤讲得非常非常清楚,一口气一直到安装DashBoard前就好，安装DashBoard我们参考其他的
* 安装flannel网络插件的时候比较曲折，文档中的kube-flannel.yml文件地址需要翻墙并且有些配置已经比较老了，推荐用`https://github.com/coreos/flannel/blob/master/Documentation/kube-flannel.yml`
 
2. 安装dashboard  
参考这个 [链接](https://www.cnblogs.com/zyxnhr/p/12181721.html#_label3) `https://www.cnblogs.com/zyxnhr/p/12181721.html#_label3` 这篇安装也很详细，不过是安装集群的，我们参考他的安装dashboard就好，有一些细节需要注意一下  
    * 文档给出的dashboard配置文件recommended.yaml地址好像要翻墙才能下载到，github上面关于这个文件的地址是`https://github.com/kubernetes/dashboard/blob/master/aio/deploy/recommended.yaml` 亲测可用，里面用到的两个镜像也不用翻墙去下载  
    
    * 配置文件的Service需要修改一下，使用nodePort的方式将其端口暴露出来，具体操作文档里面有说明 
    
    * 默认的recommended.yaml配置里面关于dashboard用户的权限只是最小的，访问apiserver的时候会403，我的做法跟文档里面写的一样。先给个admin的角色，自己开发测试使用不在意这些，具体操作文档里面也有说明  
3. 

#### 参考  
1. `https://www.jianshu.com/p/70efa1b853f5` 95%的步骤都在这里面了
2. `https://www.cnblogs.com/zyxnhr/p/12181721.html#_label3` 参考他的安装dashboard过程  
3. `https://blog.csdn.net/ganxiaomao/article/details/84949535` 安装的版本比较低，但是可以以参考他的一些环境配置以及关于kubeadm安装k8s的另外一种过程  
4. `https://www.cnblogs.com/hongdada/p/11395200.html`墙外的一些镜像可以在哪里拉取可以参考这个
5. `https://github.com/fanux/sealos` 参考文档2的时候再最后才看到这个一键部署，没有实践过，先Mark一下  

