# Docker Fundamentals

## 练习环境

操作系统：openSUSE 15.3

```bash
cat /etc/os-release
```

输出结果：

```console
NAME="openSUSE Leap"
VERSION="15.3"
ID="opensuse-leap"
ID_LIKE="suse opensuse"
VERSION_ID="15.3"
PRETTY_NAME="openSUSE Leap 15.3"
ANSI_COLOR="0;32"
CPE_NAME="cpe:/o:opensuse:leap:15.3"
BUG_REPORT_URL="https://bugs.opensuse.org"
HOME_URL="https://www.opensuse.org/"
```

## Linux原语

在操作系统中，原语（primitives）是用于创建更复杂功能的基本构建块或操作。在Linux中，有几种常用的原语。包括：

- 进程（Processes）：进程是程序或应用程序的运行实例。它们是Linux中的基本工作单元，由内核管理。

- 文件（Files）：文件是在Linux中存储数据的主要方式。它们可以是文本文件、二进制文件、目录或特殊文件，如设备文件。

- 信号（Signals）：信号是进程之间或进程与内核之间通信的一种方式。它们用于通知进程事件，例如任务完成或错误发生的情况。

- 套接字（Sockets）：套接字是Linux中进程间通信的一种方式。它们允许进程在网络或本地机器上发送和接收数据。

- 线程（Threads）：线程是轻量级的进程，与其父进程共享相同的内存空间和资源。它们通常用于通过允许同时执行多个任务来提高应用程序的性能。

- 管道（Pipes）：管道是一种将一个进程的输出连接到另一个进程的输入的方式。它们允许进程以受控的方式进行通信和交换数据。

- 信号量（Semaphores）：信号量是Linux中控制对共享资源访问的一种方式。它们允许进程协调它们对共享资源的访问，如文件或内存。

## chroot

chroot使用pivot_root，以实现将*进程*的根目录更改为任何给定的目录。

a. 模拟容器

使用`chroot`命令可以在Linux系统中创建一个容器。该容器可以看作是一个虚拟的根文件系统，其中运行的进程只能访问该根文件系统中的资源。

例如，以下命令会将当前根文件系统更改为`/tmp/myroot`目录：

```bash
chroot /tmp/myroot /bin/bash
```

这条命令会启动一个新的Bash shell，该shell的根目录为`/tmp/myroot`。

b. 更改根文件系统

`chroot`命令还可用于更改进程的根文件系统。例如，我们可以使用`chroot`命令启动一个具有另一个根文件系统的进程，而不是系统的默认根文件系统。

例如，以下命令会将当前目录（即`./`）作为根目录，并在其中启动一个新的Bash shell：

```bash
sudo chroot . /bin/bash
```

## 命名空间

在Linux操作系统中，Namespace（命名空间）是一种机制，用于隔离不同进程的资源。通过Namespace机制，可以将一组进程及其子进程的视图隔离在一个独立的Namespace中，从而实现进程之间资源隔离的目的。

下面是一些常见的Namespace类型及其作用：

1. Mount Namespace：隔离文件系统挂载点。可以使不同进程拥有自己的独立的文件系统视图。

2. PID Namespace：隔离进程ID号。可以使不同进程拥有自己的进程ID号空间，避免进程之间的PID冲突。

3. Network Namespace：隔离网络栈。可以使不同进程拥有自己的独立的网络栈，从而避免进程之间的网络冲突。

4. IPC Namespace：隔离进程间通信（IPC）机制。可以使不同进程拥有自己的独立的IPC空间，从而避免IPC机制带来的资源竞争。

5. UTS Namespace：隔离主机名和域名。可以使不同进程拥有自己的独立的主机名和域名空间，从而避免进程之间的命名冲突。

Primitives namespace和Namespace是两个不同的概念。

Namespace是Linux操作系统提供的一种机制，用于隔离不同进程的资源，以实现进程之间的资源隔离和环境隔离。例如，PID Namespace可以使不同进程拥有自己的独立的PID号空间，避免进程之间的PID冲突；Network Namespace可以使不同进程拥有自己的独立的网络栈，从而避免进程之间的网络冲突等。

Primitives namespace是一种新的技术概念，它是指将不同的基本操作（例如读写文件、创建进程、网络通信等）作为原语进行隔离和封装，使得应用程序可以在这些隔离的原语上构建出自己的隔离环境。例如，可以通过隔离文件系统读写操作来实现容器级别的文件系统隔离；通过隔离网络通信操作来实现容器级别的网络隔离等。

因此，Namespace和Primitives namespace是两个不同的概念，虽然它们都可以用于实现隔离和封装的功能，但是Namespace是一种更为通用和底层的机制，Primitives namespace是一种更为高层的抽象概念，通常用于构建容器等应用级别的隔离环境。

Namespace示例：

在Linux系统中，可以使用PID Namespace来隔离进程ID号空间，避免进程之间的PID冲突。下面是一个简单的示例：

```bash
# 创建一个新的PID Namespace
unshare -p /bin/bash

# 在新的PID Namespace中运行一个进程
echo $$ # 显示当前进程的PID
ps aux # 显示当前进程及其子进程
```

在上面的示例中，`unshare -p`命令创建了一个新的PID Namespace，并在其中启动了一个新的bash进程。由于该进程运行在一个独立的PID Namespace中，因此它的PID号与主机上的其他进程不会冲突。在这个新的bash进程中，`$$`命令显示的是该进程在PID Namespace中的PID号，而`ps aux`命令只会显示当前PID Namespace中的进程，不会显示主机上的其他进程。

Primitives Namespace示例：

在Docker容器中，可以使用Filesystem Namespace来隔离文件系统，使得不同的容器之间拥有独立的文件系统视图。下面是一个简单的示例：

```bash
# 在容器中运行一个命令
docker run --rm -it --name mycontainer ubuntu bash

# 在容器中创建一个文件并退出
touch myfile
exit

# 在主机上查看文件
ls myfile # myfile文件不存在

# 再次进入容器
docker start -i mycontainer

# 在容器中查看文件
ls myfile # myfile文件存在
```

在上面的示例中，`docker run`命令启动了一个新的Docker容器，并在其中运行了一个bash进程。由于该容器使用了Filesystem Namespace，因此容器内的文件系统视图与主机上的文件系统视图是隔离的。在容器内创建的文件`myfile`只存在于容器内部，在主机上是看不到的。当再次进入容器时，`myfile`文件就可以被看到了。

总结：

Namespace是Linux内核提供的机制，而Primitives Namespace则是一种基于Namespace的高层抽象，用于实现应用级别的隔离和封装。Namespace可以用于隔离多种资源，而Primitives Namespace通常用于隔离文件系统、网络、进程等操作的原语。

## 控制组

cgroup，全称为Control Group，即控制组，是Linux内核提供的一种机制，用于限制、记录、隔离和优先级控制一组进程的资源使用。它可以限制进程组的CPU、内存、磁盘、网络等资源的使用，同时也可以记录进程组的资源使用情况和行为。

cgroup通过将一组进程组织成一个层次结构，将资源分配给不同的cgroup来实现资源限制和优先级控制。每个cgroup可以设置资源限制和控制策略，例如可以限制一个进程组最多使用50%的CPU时间，或者限制一个进程组最多使用100MB的内存等。

cgroup最初由Google公司开发，后来被Linux内核社区采纳并加入到内核中，成为Linux系统的一部分。它在容器技术、虚拟化、云计算等领域都有广泛的应用。

下面是cgroup 的一些常见用途：

1. CPU 限制：使用 cgroup 可以限制进程的 CPU 使用率，避免某个进程占用过多的 CPU 资源导致系统负载过高，从而影响系统稳定性和其他进程的正常运行。

2. 内存限制：使用 cgroup 可以限制进程的内存使用量，避免某个进程占用过多的内存资源导致系统内存不足，从而影响系统性能和其他进程的正常运行。

3. IO 限制：使用 cgroup 可以限制进程的 IO 带宽，避免某个进程占用过多的 IO 资源导致其他进程的 IO 操作受到影响，从而影响系统性能和响应速度。

4. 网络限制：使用 cgroup 可以限制进程的网络带宽，避免某个进程占用过多的网络资源导致网络拥塞，从而影响系统性能和其他进程的正常运行。

5. 进程控制：使用 cgroup 可以限制进程的启动、停止和调度等行为，从而实现对系统进程的控制和管理。

6. 资源统计：使用 cgroup 可以实时统计系统中各个进程的资源使用情况，从而帮助管理员了解系统负载状况和各个进程的性能瓶颈，从而采取相应的措施优化系统性能。

示例：

cgroup-tools是一个用于管理Linux控制组（cgroup）的工具集，可以通过该工具集来监控和限制进程的资源使用。

在新的Systemd v248版本，原有的cgroup-tools软件包已经被废弃，推荐使用Systemd自带的cgroup工具来管理cgroup。

查看Linux系统中安装的Systemd版本，可以使用以下命令：

```bash
systemctl --version
```

执行后，会输出Systemd的版本信息，例如：

```console
#openSUSE 15.4
systemd 249 (249.12+suse.135.g7b70d88264)

#Ubuntu 22.04
systemd 250 (250-6.el9_0)

#Rocky 9.0
systemd 249 (249.11-0ubuntu3.6)
```

其中，249是Systemd的主版本号，249.12是具体的版本号。注意，Systemd的版本号可能因为发行版的不同而有所不同。

还可以使用以下命令来查看Systemd的版本信息：

```bash
rpm -q systemd
```

执行后，会输出Systemd的版本信息，例如：

```console
#openSUSE
systemd-249.12-150400.8.10.1.x86_64
#Rocky 9
systemd-250-6.el9_0.x86_64
```

Systemd自带的cgroup工具是一个命令行工具，可以使用它来创建、删除、查看和管理系统中的cgroup。以下是一些常用的命令示例：

## Apparmor和SELinux配置文件

- 安全配置文件，用于控制对资源的访问

## 内核能力

内核能力（Kernel capabilities）

- 没有能力：root可以执行所有操作，其他用户可能什么也做不了
- 38个细粒度的功能来控制权限

## seccomp策略

seccomp策略

- 限制允许的内核系统调用
- 不允许的系统调用会导致进程终止

## Netlink

Netlink

- 用于Linux内核与用户空间进程之间以及不同用户空间进程之间的进程间通信（IPC）的Linux内核接口 

## Netfilter

Netfilter

- Linux内核提供的框架，允许各种与网络相关的操作
- 包过滤、网络地址转换和端口转换（iptables/nftables）
- 用于将网络数据包定向到单个容器

更多信息可以参考 [LXC/LXD](https://linuxcontainers.org/)。

Let's download an image `alpine` to simulate an root file system under `/opt/test` folder.

```bash
mkdir test
cd test
wget https://dl-cdn.alpinelinux.org/alpine/v3.13/releases/x86_64/alpine-minirootfs-3.13.4-x86_64.tar.gz
tar zxvf alpine-minirootfs-3.13.4-x86_64.tar.gz -C alpine-minirootfs/
```

Current directory structure.

```console
tree ./test -L 1
```

Output

```
./test
├── alpine-minirootfs-3.13.4-x86_64.tar.gz
├── bin
├── dev
├── etc
├── home
├── lib
├── media
├── mnt
├── opt
├── proc
├── root
├── run
├── sbin
├── srv
├── sys
├── tmp
├── usr
└── var
```

Mount folder `/opt/test/proc` to a file and use command `unshare` to build a guest system.

```console
sudo mount -t tmpfs tmpfs /opt/test/proc
```

```console
sudo unshare --pid --mount-proc=$PWD/test/proc --fork chroot ./test/ /bin/sh
/ # ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    2 root      0:00 ps -ef
/ # touch 123
/ # ls 123
123
```

The file `123` created in guest system is accessable and writable from host system.

```console
su -
ls 123
echo hello > 123
```

We will see above change in guest system.

```console
/ # cat 123
hello
```

Let's create two folders `/opt/test-1` and `/opt/test-2`.

```console
mkdir test-1
mkdir test-2
```

Create two guests system. Mount `/opt/test/home/` to different folders for different guests.

```console
sudo mount --bind /opt/test-1 /opt/test/home/
sudo unshare --pid --mount-proc=$PWD/test/proc --fork chroot ./test/ /bin/sh
/ # cd /home
/home # echo "test-1" > 123.1
/home # cat 123.1
test-1
```

```console
sudo mount --bind /opt/test-2 /opt/test/home/
sudo unshare --pid --mount-proc=$PWD/test/proc --fork chroot ./test/ /bin/sh
/ # cd /home
/home # echo "test-2" > 123.2
/home # cat 123.2
test-2
```

```console
ll test/home
ll test-1/
ll test-2/
```

With above demo, the conclusion is that two guests share same home folder on host system and will impact each other.

## Installing Docker

Install Docker engine by referring the [guide](https://docs.docker.com/engine/), and Docker Desktop by referring the [guide](https://docs.docker.com/desktop/).

Install engine via openSUSE repository automatically.

```console
sudo zypper in docker
```

The docker group is automatically created at package installation time. 
The user can communicate with the local Docker daemon upon its next login. 
The Docker daemon listens on a local socket which is accessible only by the root user and by the members of the docker group. 

Add current user to `docker` group.

```console
sudo usermod -aG docker $USER
```

Enable and start Docker engine.

```console
sudo systemctl enable docker.service 
sudo systemctl start docker.service 
sudo systemctl status docker.service
```

## Container lifecycle

### Overview

Pull down below images in advance.

```console
docker image pull busybox
docker image pull nginx
docker image pull alpine
docker image pull jenkins/jenkins:lts
docker image pull golang:1.12-alpine
docker image pull golang
```

Download some docker images.
Create and run a new busybox container interactively and connect a pseudo terminal to it.
Inside the container, use the top command to find out that `/bin/sh` is running as process with the PID 1 and `top` process is also running. 
After that, just exit.

```console
docker image ls
docker run -d -it --name busybox_v1 -v /opt/test:/docker busybox:latest /bin/sh
docker container ps -a
docker exec -it 185efe490507 /bin/sh
/ # top
Mem: 3627396K used, 12731512K free, 10080K shrd, 2920K buff, 2999340K cached
CPU:  0.0% usr  0.1% sys  0.0% nic 99.8% idle  0.0% io  0.0% irq  0.0% sirq
Load average: 0.38 1.09 1.29 2/277 14
  PID  PPID USER     STAT   VSZ %VSZ CPU %CPU COMMAND
    1     0 root     S     1332  0.0   1  0.0 /bin/sh
    8     0 root     S     1332  0.0   2  0.0 /bin/sh
   14     8 root     R     1328  0.0   1  0.0 top
/ # exitbuild 
```

Start a new nginx container in detached mode.
Use the `docker exec` command to start another shell (`/bin/sh`) in the nginx container. 
Use ps to find out that `sh` and `ps` commands are running in your container.

```console
docker run -d -it --name nginx_v1 -v /opt/test:/docker nginx:latest /bin/sh
docker container ps -a
docker exec -it edb640127a0d /bin/sh
# ps
/bin/sh: 2: ps: not found
# apt-get update && apt-get install -y procps
# ps
   PID TTY          TIME CMD
     8 pts/1    00:00:00 sh
   351 pts/1    00:00:00 ps
# exit
```

Now we have two running containers below.

```console
docker container ps -a
```

Let's use `docker logs` to display the logs of the container we just exited from. 
The option `--since 35m` means display log in last 35 minutes.

```console
docker logs nginx_v1 --details --since 35m
docker logs busybox_v1 --details --since 35m
```

Let's make use of this to create a new stage:

Use the `docker stop` command to end your nginx container.

```console
docker stop busybox_v1
docker stop nginx_v1 
docker container ps -a
```

With above command `docker container ps -a`, we get a list of all running and exited containers. 
Remove them with docker rm.
Use `docker rm $(docker ps -aq)` to clean up all containers on your host. Use it with caution!

```console
docker rm busybox_v1
docker container ps -a
```

### Ports and volumes

Now, let's run an nginx webserver in a container and serve a website to the outside world.

Start a new nginx container and export the port of the nginx webserver to a random port that is chosen by Docker. 

Use command `docker ps` to find you which port the webserver is forwarded. Access the docker with the forwarded port number on host `http://localhost:<port#>`.

```console
docker container ps -a
docker run -d -P --name nginx_v2 nginx:latest
docker container ps -a
```

Start another nginx container and expose port to `1080` on host as an example via `http://localhost:1080`.

```console
docker run -d -p 1080:80 --name nginx_v3 nginx:latest
docker container ps -a
```

Let's make use of this to create a new stage:

Use command `docker inspect` to find out which port is exposed by the image. Network information (ip, gateway, ports, etc.) is part of the output JSON format.

```console
docker inspect nginx_v3 
```

Create a file `index.html` in folder `/opt/test` with below sample content. 

    <html>
    <head>
        <title>Sample Website from my container</title>
    </head>
    <body>
        <h1>This is a custom website.</h1>
        <p>This website is served from my <a href="http://www.docker.com" target="_blank">Docker</a> container.</p>
    </body>
    </html>

Start a new container that bind-mounts host directory `/opt/test` to container directory `/usr/share/nginx/html` as a volume, so that NGINX will publish the HTML file wee just created instead of its default message via `http://localhost:49159/` below.

```console
docker run -d -P --mount type=bind,source=/opt/test/,target=/usr/share/nginx/html --name nginx_v3-1 nginx:latest
docker container ps -a
```

Check nginx config file on where is the html home page stored in container.

```console
docker exec -it nginx_v3-1 /bin/sh
# cd /etc/nginx/conf.d
# ls
default.conf
# cat default.conf
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;  <--
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
# cd /usr/share/nginx/html
# cat index.html              
  <html>
  <head>
      <title>Sample Website from my container</title>
  </head>
  <body>
      <h1>This is a custom website.</h1>
      <p>This website is served from my <a href="http://www.docker.com" target="_blank">Docker</a> container.</p>
  </body>
  </html>
# 
```

It's recommendable to add a persistence with volumes API, instead of storing data in a docker container. Docker supports 2 ways of mount:

* Bind mounts: 
  * mount a local host directory onto a certain path in the container. 
  * Everything that was present before in the target directory is hidden (nature of the bind mount). 
  * For example, if you have some configuration you want to inject, write your config file, store it on your docker host at `/home/container/config` and mount the content of this directory to `/usr/application/config` (assuming the application reads config from there). 
  * Command: `docker run --mount type=bind,source=<source path>,target=<container path> …`
* Named volumes: 
  * docker can create a separated storage volume. 
  * Its lifecycle is independent from the container but still managed by docker. 
  * Upon creation, the content of the mount target is merged into the volume. 
  * Command: `docker run --mount source=<vol name>,target=<container path> …`

How to differentiate between bind mountbuild s and named volumes? 

* When specifying an absolute path, docker assumes a bind mount. 
* When you just give a name (like in a relative path “config”), it will assume a named volume and create a volume “config”.
* Note: Persistent storage is 'provided' by the host. It can be a part of the file system on the host directly but also an NFS mount. 

### Dockerfile

Let's build an image with a Dockerfile,build  tag it and upload it to a registry. 

Get docker image build history.

```console
docker image history nginx:latest 
```

Create an empty directory `/opt/tmp-1`, change to the directory and create an sample `index.html` file in `/opt/tmp-1`.

```console
/opt/tmp-1> cat index.html 
  <html>
  <head>
      <title>Sample Website from my container</title>
  </head>
  <body>
      <h1>This is a custom website.</h1>
      <p>This website is served from my <a href="http://www.docker.com" target="_blank">Docker</a> container.</p>
  </body>
  </html>
```

Use `FROM` to extend an existing image, specify the release number.

Use `COPY` to copy a new default website into the image, e.g., `/usr/share/nginx/html`

Create SSL configuration `/opt/tmp-1/ssl.conf` for nginx.

    server {
        listen       443 ssl;
        server_name  localhost;
    
        ssl_certificate /etc/nginx/ssl/nginx.crt;
        ssl_certificate_key /etc/nginx/ssl/nginx.key;
    
        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }
    }

Use OpenSSL to create a self-signed certificate so SSL/TLS to work would work.

Use the following command to create an encryption key and a certificate.

```console
openssl req -x509 -nodes -newkey rsa:4096 -keyout nginx.key -out nginx.crt -days 365 -subj "/CN=$(hostname)"
```

To enable encrypted HTTPS, we need to expose port 443 with the EXPOSE directive. The default nginx image only exposes port 80 for unencrypted HTTP.

In summary, we create below Dockerfile in foder `/opt/tmp-1`. 

```console
cat Dockerfile
```

Output

```
FROM nginx:latest

# copy the custom website into the image
COPY index.html /usr/share/nginx/html

# copy the SSL configuration file into the image
COPY ssl.conf /etc/nginx/conf.d/ssl.conf

# download the SSL key and certificate into the image
COPY nginx.key /etc/nginx/ssl/
COPY nginx.crt /etc/nginx/ssl/

# expose the HTTPS port
EXPOSE 443
```

We have five files in foder `/opt/tmp-1` till now.

```console
ls /opt/tmp-1
```

Output

```
Dockerfile  index.html  nginx.crt  nginx.key  ssl.conf
```

Now let's use the `docker build` command to build the image, forward the containers ports 80 and 443.

```console
docker build -t nginx:my1 /opt/tmp-1/
docker image ls

docker run -d -p 1086:80 -p 1088:443 --name nginx_v5 nginx:my1

docker container ps -a
```

Above changes can be validated via below links:

    http://localhost:1086/
    https://localhost:1088/

Register an account in [DockerHub](https://hub.docker.com/) and enable access token in Docker Hub for CLI client authentication.

```console
docker login
```

Input username and password.

```
Username: <your account id>
Password: <token>
```

Tag the image to give image a nice name and a release number as tag, e.g., name is `secure_nginx_0001`, tag is `v1`.

```console
docker tag nginx:my1 <your account id>secure_nginx_0001:v1
docker push <your account id>secure_nginx_0001:v1
docker image ls
```

### Multi-stage Dockerfile

Let's show an example of multi-stage build. The multi-stage in the context of Docker means, we can have more than one line with a FROM keyword. 

Create folder `/opt/tmp-2` and `/opt/tmp-2/tmpl`. 

Create files [edit.html](../../assets/edit.html), [view.html](../../assets/view.html), [wiki.go](../../assets/wiki.go) and structure likes below.

```
tree -l /opt/tmp-2
```

```
.
├── tmpl
│   ├── edit.html
│   └── view.html
└── wiki.go
```

Create an new Dockerfile that starts 

```console
cat Dockerfile
```

```
# app builder stage
FROM golang:1.12-alpine as builder

## copy the go source code over and build the binary
WORKDIR /go/src
COPY wiki.go /go/src/wiki.go
RUN go build wiki.go

# app exec stage
# separate & new image starts here!#
FROM alpine:3.9

# prepare file system etc
RUN mkdir -p /app/data /app/tmpl && adduser -S -D -H -h /app appuser
COPY tmpl/* /app/tmpl/

# get the compiled binary from the previous stage
COPY --from=builder /go/src/wiki /app/wiki

# prepare runtime env
RUN chown -R appuser /app
USER appuser
WORKDIR /app

# expose app port & set default command
EXPOSE 8080
CMD ["/app/wiki"]
```

Build the images by Dockerfile we created above.

```console
docker build -t lizard/golang:my1 /opt/tmp-2/
```

Run the image in detached mode, create a port forwarding from port 8080 in the container to port 1090 on the host.

```console
docker run -d -p 1090:8080 --name golan_v1 lizard/golang:my1
```

Access the container via link http://localhost:1090

Tab the golang image we created and push it to DockerHub.

```console
docker tag lizard/golang:my1 <your acccount id>/golang_0001:v1
docker push <your acccount id>/golang_0001:v1
```
