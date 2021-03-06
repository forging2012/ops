# docker深入2-docker-ce的最佳实践

2019/1/22

> 注1：凡是本人整理的，开源产品相关的文章中，标题党写明了“最佳实践”的文章，要特别注意，本人总结的文字并未涉及安全方面的指导，请参考官方的指导教程，因为安全是一个有深度的话题，且安全是相对而言的，并不是个容易的话题。

> 注2：本人整理的所有知识库，基础内容占比多，因为在学习的路上，总是容易卡在某个点上，希望能对路过的你有点帮助即可，力求普及知识，而非教科书一般的按步骤12345来指导即可上生产环境，请自行总结，走出自己的路，加油。


## 目标

部署 docker-ce 服务的最佳实践（持续更新）

部分童鞋对 docker 的版本不太理解，以下是我记忆中的事，docker 版本大致是这样演变的：
- v1.11 时代是 swarmkit
- v1.12 开始出现了 swarm mode
- v1.13 后 docker 变成 EE 和 CE 版本，CE 对应的是社区版本，并将社区的 docker 重命名为 moby，版本跳跃为：v17.03
- 当前最新版本是：v18.06

在[这里](https://github.com/docker/docker-ce/releases)查看版本


## 准备 docker 服务

### 1. 安装

```bash
# yum仓库配置
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum makecache fast

# 安装当前最新的版本
yum -y install docker-ce

# 在这里找到可用的版本：
# https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/
# (可选)指定版本
yum -y install docker-ce-18.03.1.ce-1.el7.centos.x86_64
# (可选)升级版本
yum -y upgrade docker-ce-18.03.1.ce-1.el7.centos.x86_64

# 开机启动
systemctl enable docker

# 启动服务
systemctl start docker

# 重启服务
systemctl daemon-reload
systemctl restart docker
```


### 2. 配置文件

推荐配置如下内容：
- 存储驱动
- 日志
- 阿里云registry-mirrors（可选，适用于访问 docker hub 很慢的场景，使用注意事项请参考后续FAQ#1）


有多种方式来配置 docker 服务，建议使用下述配置文件：
`/etc/docker/daemon.json`

linux 上默认没有配置文件，需要创建：
`mkdir -p /etc/docker`


### 3. 存储驱动的选择

#### 3.1 overlay 驱动(以 CentOS 最为典型)

```bash
tee /etc/docker/daemon.json <<-'EOF'
{
  "graph": "/mnt/docker-data",
  "storage-driver": "overlay",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "registry-mirrors": ["https://xxxx.mirror.aliyuncs.com"]
}
EOF
```

> 注：针对 自定义存储目录，使用 "graph": "/mnt/docker-data"
[参考文档](https://docs.docker.com/engine/admin/systemd/#start-automatically-at-system-boot)



#### 3.2 overlay2 驱动(文件存储，以 Ubuntu 最为典型)

```bash
tee /etc/docker/daemon.json <<-'EOF'
{
  "graph": "/mnt/docker-data",
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "registry-mirrors": ["https://xxxx.mirror.aliyuncs.com"]
}
EOF
```

> 注：针对 centos 系统和 docker-ce 版本，使用 `overlay2.override_kernel_check=true`
[参考文档](https://docs.docker.com/engine/userguide/storagedriver/overlayfs-driver/#configure-docker-with-the-overlay-or-overlay2-storage-driver)





#### 3.3 DeviceMapper 驱动(块存储)

```bash
yum install -y yum-utils device-mapper-persistent-data lvm2

tee /etc/docker/daemon.json <<-'EOF'
{
  "storage-driver": "devicemapper",
  "storage-opts": [
    "dm.directlvm_device=/dev/sdb",
    "dm.thinp_percent=95",
    "dm.thinp_metapercent=1",
    "dm.thinp_autoextend_threshold=80",
    "dm.thinp_autoextend_percent=20",
    "dm.directlvm_device_force=false"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "registry-mirrors": ["https://xxxx.mirror.aliyuncs.com"]
}
EOF
```

> 注：针对 让 docker 服务来管理 （自动创建） DIRECT-LVM 设备，使用 dm 相关的参数
[参考文档](https://docs.docker.com/engine/userguide/storagedriver/device-mapper-driver/#configure-direct-lvm-mode-for-production)


### 4. 配置其他特性

#### 4.1 激活 metrics 功能

```bash
# cat /etc/docker/daemon.json
{
  "metrics-addr" : "0.0.0.0:9323",
  "experimental" : true,
  (omited..)
}

# curl -s 127.0.0.1:9323/metrics
(omited..)
```


#### 4.2 定义 cgroupdriver

```bash
# cat /etc/docker/daemon.json
{
  (omited..)
  "exec-opts": ["native.cgroupdriver=cgroupfs"],
  (omited..)
}
```


### 5. 完整的配置示例

#### 5.1 实例 - centos7
```bash
# cat /etc/docker/daemon.json
{
  "metrics-addr" : "0.0.0.0:9323",
  "experimental" : true,
  "exec-opts": ["native.cgroupdriver=cgroupfs"],
  "graph": "/data/docker",
  "storage-driver": "overlay",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "registry-mirrors": ["https://xxx.mirror.aliyuncs.com"]
}
```


## swarm mode 集群常用操作示范

### 1. 网络
#### 如何在 swarm mode 中使用自定义的网络

用途：处于同一个 overlay 网络中的服务，互相之间可以通过服务名称来访问
```
docker network create --driver overlay my-network
docker service create \
    --name t001 \
    --with-registry-auth \
    --detach=true \
    --network=my-network \
    --publish 11111:80 \
    your-private-registry/your-ns/whoami:0.9
```

#### 自定义 docker_gwbridge 的网段

```bash
gwbridge_users=$(docker network inspect --format '{{range $key, $val := .Containers}} {{$key}}{{end}}' docker_gwbridge | \
xargs -d' ' -I {} -n1 docker ps --format {{.Names}} -f id={})

echo "$gwbridge_users" | xargs docker stop

docker network rm docker_gwbridge

docker network create  \
--subnet=172.18.0.1/16    \
--gateway 172.18.0.1   \
-o com.docker.network.bridge.enable_icc=false \
-o com.docker.network.bridge.name=docker_gwbridge \
docker_gwbridge
```


### 2. 格式化输出

#### 使用 go 模版来获取指定内容

例如，获取所有 overlay 网络的 subnet 信息
```bash
docker network inspect $(docker network ls -f driver='overlay' -q) \
--format='{{.Name}} -> {{if .IPAM.Config}}{{(index .IPAM.Config 0).Subnet}}{{else}}null{{end}}' |sort -k2
```


### 3. 管理 - node

#### 使用 `labels` 和 `constraint` 来调度容器

* 给 node 打标签
```bash
##### 增加标签
docker node update --label-add 'deploy.env=ops' worker1
docker node update --label-add 'deploy.env=dev' worker2
docker node update --label-add 'deploy.env=dev' worker3
docker node update --label-add 'deploy.env=qa' worker4
docker node update --label-add 'deploy.env=qa' worker5

for i in `seq 1 5`; do
  docker node inspect -f "{{.Description.Hostname}} -> {{.Spec.Labels}}" worker$i
done

##### 移除标签
docker node update --label-rm 'deploy.env' worker5
```

#### 移除 swarm node 的姿势

```bash
# 请参考如下顺序来执行，否则可能会有多个 node id 信息遗留
# curl -s 127.0.0.1:9323/metrics |grep 'swarm_node_info' 可以检查
[管理节点]# docker node demote vuwwcanp1ma14g8m3wmlp5umj
[被移除的节点]# docker swarm leave
[被移除的节点]# systemctl restart docker
[管理节点]# docker node rm vuwwcanp1ma14g8m3wmlp5umj
```


### 4. 管理 - service

#### 调度策略(Constraint)

```bash
##### 查看
for h in $(docker service ls |grep 'dev-' |awk '{print $2}' |sort); do
  echo "[+] 查看 $h 的调度策略"
  docker service inspect -f "{{.Spec.TaskTemplate.Placement.Constraints}}" $h |grep 'node.labels.deploy.env' \
  && echo '' \
  || echo 'x'
done

##### 更新
for h in $(docker service ls |grep 'dev-' |awk '{print $2}' |sort); do
  echo "[+] 更新 $h 的调度策略"
  docker service inspect -f "{{.Spec.TaskTemplate.Placement.Constraints}}" $h |grep 'node.labels.deploy.env' \
  && echo '' \
  || docker service update --with-registry-auth --detach=false --constraint-add "node.labels.deploy.env==test" $h
done

##### 移除
docker service update --with-registry-auth --detach=false --constraint-rm "node.labels.deploy.env==dev" svc1
```

#### 资源限制(Resource Limit)

```bash
# 查看
for h in $(docker service ls |grep 'dev-' |awk '{print $2}' |sort); do
  echo "[+] 查看 $h 的资源限制"
  docker service inspect -f "NanoCPUs={{.Spec.TaskTemplate.Resources.Limits.NanoCPUs}}, MemoryBytes={{.Spec.TaskTemplate.Resources.Limits.MemoryBytes}}" $h
done

# 更新
for h in $(docker service ls |grep 'dev-' |awk '{print $2}' |sort); do
  echo "[+] 更新 $h 的资源限制"
  docker service update --with-registry-auth --detach=false --limit-cpu 0.75 --limit-memory 500m $h
done
```

#### 环境变量(Env)

```bash
# 查看
for h in $(docker service ls |grep 'dev-' |awk '{print $2}' |sort); do
  echo "[+] 查看 $h 的环境变量"
  docker service inspect -f "{{.Spec.TaskTemplate.ContainerSpec.Env}}" $h
done

# 更新
for h in $(docker service ls |grep 'dev-' |awk '{print $2}' |sort); do
  echo "[+] 更新 $h 的环境变量"
  docker service update --with-registry-auth --detach=false --env-add foo=bar $h
done

# 移除
for h in $(docker service ls |grep 'dev-' |awk '{print $2}' |sort); do
  echo "[+] 移除 $h 的环境变量"
  docker service update --with-registry-auth --detach=false --env-rm foo $h
done
```


#### 容器标签(container labels)

```bash
# 查看
for h in $(docker service ls |grep 'dev-' |awk '{print $2}' |sort); do
  echo "[+] 查看 $h 的标签"
  docker service inspect -f "{{.Spec.TaskTemplate.ContainerSpec.Labels}}" $h
done

# 更新
for h in $(docker service ls |grep 'dev-' |awk '{print $2}' |sort); do
  echo "[+] 更新 $h 的标签"
  svc_name=$(echo $h |awk -F'-' '{print $NF}')
  docker service update --with-registry-auth --detach=false \
    --container-label-add "aliyun.logs.demo-stdout=stdout" \
    --container-label-add "aliyun.logs.demo-file=/data/server/demo/logs/${svc_name}_*.log" $h
done

# 移除
for h in $(docker service ls |grep 'dev-' |awk '{print $2}' |sort); do
  echo "[+] 移除 $h 的标签"
  docker service update --with-registry-auth --detach=false \
    --container-label-rm "aliyun.logs.demo-stdout" \
    --container-label-rm "aliyun.logs.demo-file" $h
done
```

#### 更新 service 的镜像

```bash
docker service update --with-registry-auth --detach=false portainer-agent --image portainer/agent:1.2.0
docker service update --with-registry-auth --detach=false portainer --image portainer/portainer:1.20.0
docker service ls |grep portainer
```

#### 查看 service 错误信息

```bash
docker service ps --no-trunc --format "{{.Node}}: {{.Error}}" dev-demo
```



## FAQ

### 在 swarm mode 中使用私有镜像仓库时遇到的问题

image xxx could not be accessed on a registry to record its digest. Each node will access xxx independently, possibly leading to different nodes running different versions of the image.

执行下面的指令时：
```
docker service create
docker service update
```
要激活这个参数来传递私有仓库的信息到 worker 节点上：
`--with-registry-auth`


### 在 swarm mode 中挂载的数据卷的目录中存在软链接目录时遇到的问题

在容器中，访问挂载卷中的路径，出现如下异常的场景：
```bash
##### 创建 service 的指令：
docker service create \
    --name demo1 \
    --with-registry-auth \
    --detach=true \
    --mount type=bind,src="/data/logs",dst="/var/log/demo1" \
    ...

##### 在容器中访问日志路径时：
~]# docker exec -it logs-agent ls /host/data/logs/
ls: /host/data/logs/: No such file or directory

##### 在容器中检查发现，上述路径在 host 上其实是一个软链接，挂载到容器中是无法访问的
~]# docker exec -it logs-agent ls -l /data/ |grep logs
lrwxrwxrwx  1 demo1  demo1    11 Jul 30 10:43 logs -> /data1/logs
```

解决方案：
请注意检查挂载的数据卷中是否存在软链接，可替换为完整的真实路径。


### 使用阿里云 Docker Hub 镜像站点和 latest 这个 tag 带来的问题

> 注1：拉取镜像时，不推荐使用 latest 这个 tag 来拉取，否则可能拉取到旧的镜像，建议指定明确的版本号。

问题描述
我的 docker node 使用了 [Docker Hub 镜像站点](https://cr.console.aliyun.com/#/accelerator)，然后发现一个问题：

````bash
~]$ sudo cat /etc/docker/daemon.json |grep registry
  "registry-mirrors": ["https://zvz7k9mh.mirror.aliyuncs.com"]

~]$ sudo docker pull opera443399/whoami
Using default tag: latest
latest: Pulling from opera443399/whoami
Digest: sha256:4119c322c2d9f007af8751394a63ed6809158bc39456e6aa60ab84e7a21429c5
Status: Image is up to date for opera443399/whoami:latest

~]$ sudo docker images |grep opera
opera443399/whoami         latest              160ed79ce86f        5 weeks ago         4.13MB
````

实际上，上述拉取到的是一个旧的版本 160ed79ce86f

> 对比下面操作中另一个节点的状态（没有使用加速镜像）
````bash
~]$ sudo docker pull opera443399/whoami
Using default tag: latest
latest: Pulling from opera443399/whoami
Digest: sha256:a05120d9fe157868f7f1c8b842cc860fb58665d74cadaf8eb7d6091af626cccd
Status: Downloaded newer image for opera443399/whoami:latest

~]$ sudo docker images |grep opera
opera443399/whoami          0.9                  a7878d3e0fdf        20 hours ago        4.13MB
opera443399/whoami          latest               a7878d3e0fdf        20 hours ago        4.13MB
opera443399/whoami          0.8                  ae008c956c53        3 weeks ago         4.13MB
opera443399/whoami          0.7                  160ed79ce86f        5 weeks ago         4.13MB

````

小结：使用阿里云的容器服务中的 Docker Hub 镜像站点后，缓存的 latest 这个 tag 是旧版本，且过了好几天还没有刷新，这个问题怎么解决？



已经试图反馈给[阿里云](https://github.com/aliyun/aliyun-cli/issues/36)，如果进展，后续更新。


### 在 swarm mode 中 update service 时遇到异常的问题

执行 update 操作时，失败(尽管swarm会在数秒后自动再次调度资源来修正问题)，排查时得到如下报错信息：
```
"starting container failed: OCI runtime create failed: container_linux.go:348: starting container process caused "process_linux.go:301: running exec setns process for init caused \"exit status 40\"": unknown"
```

未查明具体原因，可能原因是，还在跟踪中：
```
痕迹1: docker service update 操作完成后，会有很多 'Exited' 状态的 container 遗留下来(观察时，3节点的集群存在这样的container的数量2000+)，可能占用了部分资源。
通过清理后：
docker container prune --filter "until=24h" --force
```



## 参考
1. [systemd](https://docs.docker.com/engine/admin/systemd/)
2. [storagedriver](https://docs.docker.com/engine/userguide/storagedriver)
3. [Docker CE 镜像源站](https://yq.aliyun.com/articles/110806)
4. [阿里云容器服务-镜像加速器](https://cr.console.aliyun.com/#/accelerator)
5. [Docker --format 格式化输出概要操作说明](https://yq.aliyun.com/articles/230067)
6. [How do I change the docker gwbridge address?](https://success.docker.com/article/how-do-i-change-the-docker-gwbridge-address)
7. [Docker service update --image "could not accessed on a registry to record its digest" #34153](https://github.com/moby/moby/issues/34153)
