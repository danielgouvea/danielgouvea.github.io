---
layout: post
title:  "Developing on CLion using Docker"
date:   2020-03-22 17:00:45 -0400
categories: c++ clion docker toolchain
---

It is very common nowadays that a developer do day to day coding using a Macbook but at the end the code will run on a Linux server in the cloud. You can always have a similar toolchain installed on your macOS, but you won't have the same standard library implementation, nor specific Linux headers available. Wouldn't be great to develop and test your code on your local environment using the exact Linux distro? Docker to the rescue!

Docker can provide you a containerized version of most Linux distros running on your computer as if it was a remote machine, and then you can configure CLion to work alongside it seamlessly.

## Docker

The first step is downloading Docker from its [official site](https://www.docker.com/get-started) and installing it.

### Dockerfile

Having Docker installed, we will need a `Dockerfile` describing which Linux distro do we wish to use, and the steps to build an image containing the desired toolchain and basic SSH communication.

```dockerfile
FROM ubuntu:20.04

# Install required packages (ssh and toolchain)
ENV DEBIAN_FRONTEND=noninteractive 
RUN apt-get --yes update
RUN apt-get --yes upgrade
RUN apt-get --yes install openssh-server rsync build-essential gdb cmake

# Create user named 'clion' with password 'password'
RUN useradd -m clion && yes password | passwd clion

# Create privilege separation directory
RUN mkdir /var/run/sshd

# Expose SSH port
EXPOSE 22/tcp

# Run SSH server
CMD ["/usr/sbin/sshd", "-D"]
```

In this example, we are using Ubuntu 20.04, which is the most recent LTS version. CLion will connect to the container using SSH and will need to transfer files, so the following packages are required: `openssh-server` and `rsync`. The selected toolchain is GCC, GDB and CMake, hence the packages: `build-essential`, `gdb` and `cmake`. All the other steps are configuring a user and the SSH server, but keep in mind that each distro will have its own specific commands.

### Docker Image

The next step is to build the image itself running the following docker command in the same directory as the `Dockerfile`.

```shell
docker build -t clion/ubuntu-20.04-gcc-9 .
```

The image just need to be built once, as it will be saved in the local Docker repository. In this example, we named it as `clion/ubuntu-20.04-gcc-9` to highlight the distro and compiler versions.

### Docker container

```shell
docker run --rm --detach -p 127.0.0.1:10000:22/tcp --name clion-ubuntu-20.04-gcc-9 clion/ubuntu-20.04-gcc-9
```

```shell
docker ps --all
```

```text
CONTAINER ID        IMAGE                      COMMAND               CREATED             STATUS              PORTS                     NAMES
bf17fc40dbf8        clion/ubuntu-20.04-gcc-9   "/usr/sbin/sshd -D"   7 seconds ago       Up 7 seconds        127.0.0.1:10000->22/tcp   clion-ubuntu-20.04-gcc-9
```

```shell
docker stop clion-ubuntu-20.04-gcc-9
```

```shell
docker stats
```

```text
CONTAINER ID        NAME                       CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
bf17fc40dbf8        clion-ubuntu-20.04-gcc-9   0.00%               1.398MiB / 3.846GiB   0.04%               836B / 0B           0B / 0B             1
```

## CLion


## References
https://blog.jetbrains.com/clion/2020/01/using-docker-with-clion/
https://www.jetbrains.com/help/clion/remote-development.html
https://www.jetbrains.com/help/clion/remote-projects-support.html
https://docs.docker.com/engine/examples/running_ssh_service/