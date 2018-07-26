# gitlab-runner

[Run GitLab Runner in a container](https://docs.gitlab.com/runner/install/docker.html)

## step 1：运行gitlab-runner镜像

```bash
sudo docker run \
    --name gitlab-runner \
    --net gitlab-network \
    --detach \
    --volume /srv/gitlab-runner/config:/etc/gitlab-runner \
    --volume /var/run/docker.sock:/var/run/docker.sock \
    gitlab/gitlab-runner:latest
```

这里注意`net gitlab-network`就是将`gitlab-runner`容器加入到和`gitlab`容器同一个`gitlab-network`docker网络中。

## step 2：注册gitlab-runner

```bash
docker exec -it gitlab-runner gitlab-runner register
```

```bash
Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com )
http://gitlab
```

因为在同一个docker network中所以可以直接通过docker name注册。

```bash
Please enter the gitlab-ci token for this runner
xxx
```

token可以在gitlab上用root角色登陆，在`Admin Area -> Runners`找到。

```bash
Please enter the gitlab-ci description for this runner
[hostame] my-runner
```

```bash
Please enter the gitlab-ci tags for this runner (comma separated):

```

```bash
Please enter the executor: ssh, docker+machine, docker-ssh+machine, kubernetes, docker, parallels, virtualbox, docker-ssh, shell:
docker
```

这里使用docker做作为runner的执行器，这样可以保证ci环境的干净。

```bash
Please enter the Docker image (eg. ruby:2.1):
alpine:latest
```

## step 3：修改config.toml

因为我们使用了docker作为executor，在clone仓库的时候是直接使用`git@gitlab.local.com:km/awesome.git`，这样在executor中就无法访问到gitlab.local.com这个域名。这样就要配置**config.toml**。

```bash
vim /srv/gitlab-runner/config/config.toml
```

添加：

```bash
[[runners]]
  clone_url = "http://gitlab"
  [runners.docker]
    network_mode = "gitlab-network"
    volumes = ["/cache:/cache"]
```

这样就是将clone的url改为`http://gitlab`，就是`gitlab`容器的名；并且将executor的docker加入到`gitlab-network`中，这样就可访问到`gitlab`容器中仓库了。
`volumes = ["/cache:/cache"]`可以在有多个runner的时候共用cache。

done