# Docker Fundamentals

## Demo environment

Linux: openSUSE 15.3
```console
cat /etc/os-release
```
Output
```
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

## Linux Primitives

chroot(using pivot_root)

- Changes the root directory for a process to any given directory

namespaces

- Different processes see different environments even though they are on the same host/OS
    - mnt (mount points)
    - pid (process tree)
    - net (network interfaces and connectivity)
    - ipc (interprocess communication framework)
    - uts (unix timesharing - domain name, hostname, etc.)
    - uid (user IDs and mappings)

cgroups(control groups)

- manage/limit resource allocation to individual processes
- Prioritization of processes

Apparmor and SELinux profiles
- Security profiles to govern access to resources

Kernel capabilities

- without capabilities: root can do everything, everybody else may do nothing
- 38 granular facilities to control privileges

seccomp policies

- Limitation of allowed kernel syscalls
- Unallowed syscalls lead to process termination

Netlink
- A Linux kernel interface used for inter-process communication (IPC) between both the kernel and userspace processes, and between different userspace processes. 

Netfilter

- A framework provided by the Linux kernel that allows various networking-related operations
- Packet filtering, network address translation, and port translation(iptables/nftables)
- used to direct network packages to individual containers

More inforamtion could refer to [LXC/LXD](https://linuxcontainers.org/)

Let's download an image `alpine` to simulate an root file system under `/opt/test` folder.
```console
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









