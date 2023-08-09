---
title: "Docker容器资源的占用情况"
date: 2023-08-09T21:56:23+08:00
draft: false
---

# 1、启动容器并限制资源
启动一个centos容器，限制其内存为1G ，可用cpu数为2
```bash
docker run --name os1 -it -m 1024m或者1g --cpus=2 centos:latest bash
```
# 2、查看容器状态
```bash
docker top 容器名： 查看容器的进程，不加容器名即查看所有
docker stats 容器名：查看容器的CPU，内存，IO 等使用信息

docker stats (不带任何参数选项)

docker stats --no-stream  (只返回当前的状态)

docker stats   --no-stream   容器ID/Name (只输出指定的容器)

docker stats --format  “table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}" (格式化输出的结果)

```
输出的主要了内容
```bash
[CONTAINER]：以短格式显示容器的 ID。

[CPU %]：CPU 的使用情况。

[MEM USAGE / LIMIT]：当前使用的内存和最大可以使用的内存。

[MEM %]：以百分比的形式显示内存使用情况。

[NET I/O]：网络 I/O 数据。

[BLOCK I/O]：磁盘 I/O 数据。 

[PIDS]：PID 号。
```

**获取容器的CPU消耗排行前五**
```bash
docker stats --no-stream|awk '{print $3,"                    "$1,$2}'|sort -h |tail -n 5
```

**获取容器的内存消耗排行前五个**
```bash
docker stats --no-stream|awk '{print $4,"                    "$1,$2}'|sort -h |tail -n 5
```