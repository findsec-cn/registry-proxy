# 搭建 Docker 镜像仓库代理站点

在使用 Kubernetes 时，我们需要经常访问 gcr.io 镜像仓库，由于众所周知的原因，gcr.io 在中国无法访问。gcr.azk8s.cn 是 gcr.io 镜像仓库的代理站点，原来可以通过 gcr.azk8s.cn 访问 gcr.io 仓库里的镜像，但是目前 *.azk8s.cn 已经仅限于 Azure 中国的 IP 使用，不再对外提供服务了。为了能够顺利访问 gcr.io 镜像仓库，我们需要在墙外自己搭建一个类似于 gcr.azk8s.cn 镜像仓库代理站点。

## 前提条件

- 一台能够翻墙的服务器
- 一个域名和域名相关的SSL证书（docker pull 镜像时需要验证域名证书）

## 安装配置 Docker

### 添加 Docker yum 仓库

    $ yum install -y yum-utils
    $ yum-config-manager \
    --add-repo \
    https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

### 安装 Docker

    $ yum install -y docker-ce docker-ce-cli containerd.io

### 配置 Docker

如下配置 Docker，设置 Docker 的日志格式为 json，日志文件大小为 100M，最多保存 3 个日志；接下来设置 Docker 镜像私有仓库和官方镜像加速地址；设置 Docker 的数据目录到 /data/docker；最后设置 Docker 的 Storage Driver 为 overlay2。

    $ mkdir /etc/docker
    $ cat << EOF > /etc/docker/daemon.json
    {
      "log-driver": "json-file",
        "log-opts": {
          "max-size": "100m",
          "max-file": "3"
        },
      "insecure-registry": [
        "hub.xxx.com"
      ],
      "registry-mirror": "https://q00c7e05.mirror.aliyuncs.com",
      "data-root": "/data/docker",
      "exec-opts": ["native.cgroupdriver=systemd"],
      "storage-driver": "overlay2",
      "storage-opts": [
        "overlay2.override_kernel_check=true"
      ]
    }
    EOF

### 启动 Docker

    $ systemctl enable docker && systemctl start docker

## 安装 Docker Compose

    $ curl -L "https://github.com/docker/compose/releases/download/1.25.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    $ chmod +x /usr/local/bin/docker-compose
    $ ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
    $ docker-compose --version
    docker-compose version 1.25.4, build 1110ad01

## 镜像仓库代理站点

### 启动前准备

从 github 下载 registry-proxy 配置文件：

    $ git clone https://github.com/findsec-cn/registry-proxy.git
    $ cd registry-proxy

将域名的证书放置到 cert 目录下，其中 server.crt 为 ssl 证书文件， server.key 为 ssl 私钥。

修改 nginx.conf 配置文件，将配置文件中的域名替换成自己的域名(yyy.com)：

    $ sed -i 's/xxx.com/yyy.com/g' nginx.conf

### 启动镜像仓库代理站点

启动镜像仓库代理站点：

    $ docker-compose up -d

查看启动日志：

    $ docker-compose logs -f

### 解析域名

将 hub.yyy.com、gcr.yyy.com 解析到此服务器的地址上。

我们可以通过 http://hub.yyy.com 查看镜像仓库缓存的镜像；可以通过gcr.yyy.com下载镜像。

## 使用镜像仓库代理站点

我们只需要**将 k8s.gcr.io 替换成 gcr.yyy.com/google-containers**；**将 gcr.io 替换成 gcr.yyy.com** 就可以下载 gcr.io 仓库里的镜像了。

比如我们要下载镜像：

    $ docker pull k8s.gcr.io/pause:3.1

可以如下通过镜像仓库代理站点下载：

    $ docker pull gcr.yyy.com/pause:3.1

比如我们要下载镜像：

    $ gcr.io/kubernetes-helm/tiller:v2.16.3
    $ gcr.io/google-containers/etcd:3.2.24

可以如下通过镜像仓库代理站点下载：

    $ gcr.yyy.com/kubernetes-helm/tiller:v2.16.3
    $ gcr.yyy.com/google-containers/etcd:3.2.24

如果你用 kubeadm 部署 Kubernetes 集群，可以在 kubeadm 配置文件中设置镜像地址为：gcr.yyy.com/google-containers

    $ cat kubeadm-config.yaml

    apiVersion: kubeadm.k8s.io/v1beta1
    kind: ClusterConfiguration
    kubernetesVersion: v1.18.1
    ......
    imageRepository: gcr.yyy.com/google-containers
