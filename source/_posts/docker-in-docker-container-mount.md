---
tags:
  - docker
categories:
  - blog
title: Mount volume to a docker container running in a docker container
date: 2019-05-05 21:01:39
---

# 一个匪夷所思的需求
有时候我们需要在 docker container 里面运行一个 docker container（虽然都是运行在宿主机的 docker daemon 里的），这个其实使用 `-v /var/run/docker.sock:/var/run/docker.sock` 就能做到了。本文的重点是下面这种情况：

![](1.png)

也就是说，我们想让最里层的 container 2 和中间的 container 都能够访问宿主机上的一个目录。

# 在 linux 下的实现方法
这个其实非常简单，只需要在运行 docker 的时候使用宿主机的目录即可：
```bash
~ # docker run -it --rm -v ~/your-path/:/a -v /var/run/docker.sock:/var/run/docker.sock docker
/ # ls /a
color.Dockerfile        makefile                test.Dockerfile         testArgPath.Dockerfile
/ # docker run -it --rm -v ~/your-path/:/b -v /var/run/docker.sock:/var/run/docker.sock docker
/ # ls /a
ls: /a: No such file or directory
/ # ls /b
color.Dockerfile        makefile                test.Dockerfile         testArgPath.Dockerfile
```

由于在 container 1 中调用 docker cli 是直接访问的宿主机的 docker daemon，因此可以挂载宿主机上的任意一个路径。这里要注意的是，即使是在 container 1 中运行的 docker cli，在使用 -v 挂载路径的时候指定的也是宿主机里的路径，而不是 container 中的路径。

# 在 windows 下的实现方法
(我只在 docker-desktop (linux 容器) 里实验过，不保证在其他版本里也能用。)

由于 container 1 中只能接收 linux 格式的路径，只需要把类似于 `d:/Sources` 这样的格式换成 `/d/Sources` 即可：

![](2.png)
