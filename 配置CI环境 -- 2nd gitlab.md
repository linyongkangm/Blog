# gitlab

[GitLab Docker images](https://docs.gitlab.com/omnibus/docker/)

## pre

考虑到还有gitlab-runner要安装，所以我们使用docker network建立一个docker的网络，在network内的容器就可以使用docker name通信。

```bash
docker network create gitlab-network
```

## step 1：运行gitlab镜像

```bash
sudo docker run \
    --name gitlab \
    --hostname gitlab.local.com \
    --publish 80:80 --publish 22:22 --publish 443:443 \
    --net gitlab-network \
    --detach \
    --restart always \
    --volume /srv/gitlab/config:/etc/gitlab \
    --volume /srv/gitlab/logs:/var/log/gitlab \
    --volume /srv/gitlab/data:/var/opt/gitlab \
    gitlab/gitlab-ce:latest
```

```bash
docker run gitlab/gitlab-ce:latest
```

就是运行[gitlab-ce（GitLab Community Edition）](https://registry.hub.docker.com/u/gitlab/gitlab-ce/)镜像，当然也可以使用[gitlab-ee（GitLab Enterprise Edition）](https://registry.hub.docker.com/u/gitlab/gitlab-ee/);

```bash
--name gitlab
```

将容器命名为`gitlab`

```bash
--hostname gitlab.local.com
```

对应![gitlab-hostname](https://github.com/linyongkangm/Blog/blob/master/public/images/gitlab-hostname.png)。注意，不能用IP地址~~

```bash
--publish 80:80 --publish 22:22 --publish 443:443
```

绑定端口，这里注意22端口一般都启用作ssh服务的，我因为使用虚拟机并且专门是跑gitlab相关，所以我选择kill 22端口的进程。

```bash
lsof -i:22 // 获得22端口的pid
kill pid // kill
```

当然也可以绑定不一样的端口号：

```bash
--publish 8929:80 --publish 2289:22
```

然后就要在容器的`/etc/gitlab/gitlab.rb`或宿主机的`/srv/gitlab/config/gitlab.rb`进行设置：

```bash
external_url "http://gitlab.example.com:8929"
gitlab_rails['gitlab_shell_ssh_port'] = 2289
gitlab-ctl restart
```

```bash
--net gitlab-network
```

将该容器加入到`gitlab-network`

```bash
--volume /srv/gitlab/config:/etc/gitlab \
--volume /srv/gitlab/logs:/var/log/gitlab \
--volume /srv/gitlab/data:/var/opt/gitlab \
```

文件映射，docker本身是无状态的，所以要用宿主环境的文件夹映射到容器对应的文件夹。

## step 2:访问

### 使用IP访问

理论上可以使用虚拟机的IP进行访问，但是在创建容器的时候hostname不能使用IP，所以我们可以修改配置使其生效：

```bash
vim /src/gitlab/data/gitlab-rails/etc/gitlab.yml
```

将`gitlab.host`，`gitlab.port`和`gitlab.ssh_host`修改正确：

```bash
gitlab:
    host: '虚拟机IP',
    port: 'gitlab宿主机的对外HTTP映射端口'

    ssh_host: '虚拟机IP:gitlab宿主机的对外SSH映射端口'
```

### 使用域名访问

修改`gitlab.yml`是`uncomment`的，在step1我们设定了`hostname`，所以我们可以修改主机的hosts文件一样可以访问到。

done