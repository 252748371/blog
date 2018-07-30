---
title: Docker备忘录
date: 2018-07-16 10:08:18
tags: Docker
---

> 删除全部image的话

docker rmi $(docker images -q)

> 删除全部container

docker rm $(docker ps -aq)

<!-- more -->

>boot2docker用户和密码


用户名 | 密码 | 进入方式
---|---|---
docker | tcuser | ssh
root | 空 | command：sudo -i (docker用户下执行)

> zookeeper docker运行命令

docker pull zookeeper

docker run --name zookeeper-1 --restart always -d zookeeper

docker exec -it 容器id zkCli.sh 