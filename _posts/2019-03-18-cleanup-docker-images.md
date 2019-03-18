---
title: Clean Up Your Docker Images
categories: tips docker
---
## Show All Docker Images

```shell
$ docker images
```

**Output:**

```
REPOSITORY                         TAG                 IMAGE ID            CREATED             SIZE
jemalloc                           latest              f55e72747051        47 hours ago        897MB
centos-java-slim                   8u201-b09           a0e265286b7f        12 days ago         3.55MB
aukhatov/centos-java               8u201-b09           31f71c1b4eda        12 days ago         311MB
aukhatov/srping-boot-auth-server   0.1.0               c6159d63ad1c        2 weeks ago         136MB
aukhatov/mem-eat                   1.0                 4e6aa1979b36        4 weeks ago         105MB
openjdk                            8-jdk-alpine        792ff45a2a17        5 weeks ago         105MB
centos                             7                   1e1148e4cc2c        3 months ago        202MB
postgres                           latest              f9b577fb1ed6        3 months ago        311MB
ubuntu                             latest              cd6d8154f1e1        6 months ago        84.1MB
redis                              latest              bfcb1f6df2db        10 months ago       107MB
eclipse-mosquitto                  latest              bd592a7a5bcf        14 months ago       4.38MB
```

## Get Docker Image Ids

```shell
docker images | tr -s ' ' | cut -d ' ' -f3
```

**Output:**

```
IMAGE
f55e72747051
a0e265286b7f
31f71c1b4eda
c6159d63ad1c
4e6aa1979b36
792ff45a2a17
1e1148e4cc2c
f9b577fb1ed6
cd6d8154f1e1
bfcb1f6df2db
bd592a7a5bcf
```

## Clean Up All Docker Images

And you can pass the previous command result into the `docker rmi` command.

```shell
$ docker rmi -f $(docker images | tr -s ' ' | cut -d ' ' -f3)
```

## Clean Up All Docker Container

```shell
$ docker rm $(docker ps -a | tr -s ' ' | cut -d ' ' -f1)
```