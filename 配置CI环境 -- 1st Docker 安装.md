# Docker 安装

[Docker CE 镜像源站](https://yq.aliyun.com/articles/110806?spm=5176.8351553.0.0.35501991q6DKpk)

## step 1: 安装必要的一些系统工具

```bash
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

## Step 2: 添加软件源信息

```bash
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

## Step 3: 更新并安装 Docker-CE

```bash
sudo yum makecache fast
sudo yum -y install docker-ce
```

## Step 4: 开启Docker服务

```bash
sudo service docker start
```

现在Docker就已经配置完成了，但是理所当然的，我们要用镜像加速，先要去[aliyun](https://cr.console.aliyun.com/#/accelerator)申请镜像加速器。

## Step 5: 配置镜像加速器

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://xxxxxxxxxxxxxxxxxxxxx.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## Step 6: 验证

```bash
sudo docker run hello-world
```

```bash
hello world
```

done