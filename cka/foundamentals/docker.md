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



下面是openSUSE中的示例：

安装需要的软件包：

```bash
sudo zypper install libcgroup-tools
```

限制CPU使用上限：

```bash
# 创建新的cgroup 'mygroup'
sudo mkdir /sys/fs/cgroup/cpu/mygroup

# 系统会创建默认的一些文件，含初始值，比如CPU使用时间的限额的默认值是-1
cat /sys/fs/cgroup/cpu/mygroup/cpu.cfs_quota_us

# 设定CPU使用时间上限
sudo sh -c "echo 50000 > /sys/fs/cgroup/cpu/mygroup/cpu.cfs_quota_us"

# 启动一个新的进程，并且关联到
sudo cgcreate -g cpu:mygroup
sudo cgexec -g cpu:mygroup /bin/bash
```

在上面的例子中，`cpu.cfs_quota_us` 文件设置了 cgroup 中的进程可以使用的最大 CPU 时间。该值以微秒为单位，因此将其设置为 50000 表示进程最多可以使用单个 CPU 核心的 50%。`cgcreate `和 `cgexec `命令创建并将进程`/bin/bash`移动到 `mygroup `cgroup 中。

限制内存使用上限：

```bash
# 创建新的cgroup 'mygroup'
sudo mkdir /sys/fs/cgroup/memory/mygroup

# 系统会创建默认的一些文件，含初始值，比如内存使用上限的默认值是9223372036854771712
cat /sys/fs/cgroup/memory/mygroup/memory.limit_in_bytes

# 设置内存使用上限512MB
sudo sh -c "echo 536870912 > /sys/fs/cgroup/memory/mygroup/memory.limit_in_bytes"

# 启动一个新的进程，并且关联到'mygroup'
sudo cgcreate -g memory:mygroup
sudo cgexec -g memory:mygroup /bin/bash
```

在上面例子中，`memory.limit_in_bytes` 文件设置了 cgroup 中进程可以使用的最大内存量。该值以字节为单位，因此将其设置为 536870912 表示进程最多可以使用 512MB 的内存。

设置优先进程的 I/O 使用率：

```bash
# 创建新cgroup 'mygroup'
sudo mkdir /sys/fs/cgroup/blkio/mygroup

# 设置进程最大读和写的速率10MB/s
sudo sh -c "echo '8:0 10485760' > /sys/fs/cgroup/blkio/mygroup/blkio.throttle.read_bps_device"
sudo sh -c "echo '8:0 10485760' > /sys/fs/cgroup/blkio/mygroup/blkio.throttle.write_bps_device"

# 启动一个新的进程，并且关联到'mygroup'
sudo cgcreate -g blkio:mygroup
sudo cgexec -g blkio:mygroup /bin/bash

```

在上面例子中，`blkio.throttle.read_bps_device`和`blkio.throttle.write_bps_device`文件设置了cgroup中进程可以使用的最大读取和写入带宽。该值以每秒字节数为单位，因此将其设置为10485760意味着进程在主设备号:次设备号为8:0的设备（通常是第一个硬盘）上读取或写入的带宽最多为10MB/s。

将 `8:0 10485760` 这个字符串写入到 `/sys/fs/cgroup/blkio/mygroup/blkio.throttle.read_bps_device` 文件中的作用是限制 `mygroup` 控制组中关联的块设备（block device）的读取速率。

在 Linux 中，`blkio` 控制组子系统可以用来对进程或线程的块设备访问进行限制，如限制读写速率、I/O 优先级等。而 `blkio.throttle.read_bps_device` 这个文件则用于设置某个块设备的读取速率限制。

具体来说，`8:0` 表示设备的主次编号（major:minor），这里是指磁盘 `/dev/sda`。`10485760` 则是读取速率的限制值，单位是字节/秒。这个值表示 `/dev/sda` 最大读取速率为 10MB/s，超过这个速率的读取请求会被延迟执行，从而限制了磁盘的读取带宽。

因此，以上命令的含义是将 `mygroup` 控制组中关联的 `/dev/sda` 磁盘的读取速率限制为 10MB/s，从而实现对该控制组中进程或线程对磁盘读取的限制。

同理，将 `8:0 10485760` 这个字符串写入到 `/sys/fs/cgroup/blkio/mygroup/blkio.throttle.write_bps_device` 文件中，以限制 `mygroup` 控制组中关联的块设备（block device）的写入速率。



限制一组进程的网络带宽：

```bash
# 创建新的cgroup 'mygroup'
sudo mkdir /sys/fs/cgroup/net_cls/mygroup

# 将此组中的进程的网络类ID设置为“myclass”
sudo sh -c "echo 0x10001 > /sys/fs/cgroup/net_cls/mygroup/net_cls.classid"

```

上面的例子是将 `0x10001` 这个十六进制数值写入到`/sys/fs/cgroup/net_cls/mygroup/net_cls.classid` 文件中，以指定 `mygroup` 控制组的网络类别标识符（classid）。

网络类别标识符是 Linux 内核中用来实现流量控制和流量分类的一个机制，它可以将数据包按照不同的类别（class）进行标记和区分，然后在网络设备上针对不同的类别进行不同的处理，如限速、优先级调整等。控制组中的 `net_cls` 子系统可以用来将进程或线程与网络类别标识符关联起来，从而实现对它们的网络流量进行控制和分类。

因此，以上命令是将 `mygroup` 控制组的网络类别标识符设置为 `0x10001`，这样与该控制组相关联的进程或线程就会被标记为该类别，然后可以通过其他工具（如 `tc` 命令）对其进行网络流量控制和分类。



如果遇到对应限制文件不存在，一种可能是需要检查cgroup子系有没有正确统载或者没有启用内存子系统。

```bash
mount | grep cgroup
```

如果 cgroups 文件系统已经挂载，应该会看到输出类似于以下内容（）以memory为例）：

```
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
```

如果没有看到 `memory` 字段，则表示内存子系统没有启用。可以编辑`/etc/default/grub` 文件，添加或修改以下行：

```console
GRUB_CMDLINE_LINUX_DEFAULT="cgroup_enable=memory"
```

然后更新 GRUB 配置并重启系统：

```bash
sudo update-grub
sudo reboot
```

重启后再次检查 `/sys/fs/cgroup/memory/mygroup/memory.limit_in_bytes` 文件是否存在。如果还是不存在，可能需要手动创建它以及其他相关的 cgroups 文件。例如，运行以下命令：

```bash
sudo mkdir /sys/fs/cgroup/memory/mygroup
sudo touch /sys/fs/cgroup/memory/mygroup/memory.limit_in_bytes
```

然后就可以像之前的例子一样设置内存限制了

## Apparmor和SELinux配置文件

- 安全配置文件，用于控制对资源的访问



AppArmor 和 SELinux 都是常见的强制访问控制（MAC）机制，可以对进程或应用程序的访问权限进行精细控制。下面分别举例说明这两种机制的配置文件使用。

1. AppArmor

AppArmor 的主配置文件是 `/etc/apparmor/profiles.d/` 目录下的各个文件，每个文件对应一个应用程序或进程的配置。以 `sshd` 服务为例，该服务的配置文件是`/etc/apparmor.d/usr.sbin.sshd`。

该配置文件的内容类似于下面这样：

```console
# Last Modified: Sun Mar 14 18:53:00 2023
#include <tunables/global>

/usr/sbin/sshd {
  #include <abstractions/base>
  #include <abstractions/nameservice>

  # allow read access to user home directories
  /home/** r,

  # allow sshd to execute /usr/bin/which to determine full path of shell
  /usr/bin/which ix,

  # allow sshd to read its own configuration file
  /etc/ssh/sshd_config r,

  # allow sshd to read the SSH host keys
  /etc/ssh/ssh_host_* r,

  # allow sshd to use pam for authentication
  /usr/share/pam/** r,

  # allow sshd to use nsswitch for name resolution
  /etc/nsswitch.conf r,
  /etc/hosts r,
  /etc/hostname r,
  /etc/resolv.conf r,

  # allow sshd to write to its own log file
  /var/log/auth.log w,

  # allow sshd to create and manage pid files
  /var/run/sshd.pid w,
  /var/run/sshd.dir/ w,
  /var/run/sshd.dir/* rw,

  # allow sshd to access systemd-logind
  /run/systemd/* r,
  /run/systemd/session/*.scope r,
  /run/systemd/sessions/*.scope r,

  # deny everything else
  deny /,
}


```

该配置文件定义了 `/usr/sbin/sshd` 进程的权限限制规则，包括允许访问的文件、禁止访问的文件等。其中 `#include <abstractions/base>` 表示包含了一组通用的权限规则，可以在不同的应用程序配置中重复使用。

2. SELinux

SELinux 的主配置文件是 `/etc/selinux/config`，该文件定义了系统的 SELinux 策略和模式。默认情况下，openSUSE 使用的是 `targeted` 模式。

每个进程或应用程序还需要对应一个 SELinux 配置文件，以定义它们的访问权限。以 `httpd` 服务为例，该服务的 SELinux 配置文件是`/etc/selinux/targeted/contexts/httpd.te`。

该配置文件的内容类似于下面这样：

```bash
# HTTPD server
type httpd_t;
type httpd_sys_script_t;
init_daemon_domain(httpd_t, httpd_sys_script_t)

```

该配置文件定义了 `httpd` 服务的 SELinux 类型为 `httpd_t`，并使用了`httpd_sys_script_t` 作为其初始化域。其中 `type` 表示 SELinux 类型，`init_daemon_domain` 则是一个 SELinux 宏，用于定义服务的初始域。

需要注意的是，在 SELinux 中，访问权限规则不是直接在配置文件中定义的，而是通过访问控制策略和规则进行控制。这些策略和规则可以使用 SELinux 工具集（如 `semanage`、`setsebool`、`restorecon` 等）进行管理和设置。

比如，在openSUSE中可以看到`/etc/selinux/semanage.conf`文件和其中的配置。



## 内核能力

内核能力（Kernel capabilities）

- 没有能力：root可以执行所有操作，其他用户可能什么也做不了
- 38个细粒度的功能来控制权限



Kernel capabilities 是 Linux 内核提供的一种机制，用于控制进程对系统资源的访问权限。与传统的 Unix 权限机制不同，Kernel capabilities 可以使管理员在精细控制系统资源访问的同时，避免将过多权限授予进程，提高了系统的安全性。

在传统 Unix 权限机制中，每个进程都有一个有效用户 ID 和一个有效组 ID，这些 ID 决定了该进程对文件、设备、网络等资源的访问权限。但是，这种权限机制不够灵活，如果要授予进程某些特定的权限，可能需要将所有的权限都授予给它，从而降低了系统的安全性。

Kernel capabilities 提供了一种更细粒度的权限控制方式。每个进程都有一组 capabilities，每个 capability 表示一种特定的权限。进程可以请求和释放某些 capability，这样就可以将权限授予进程，而不必授予所有权限。

例如，可以将 `CAP_NET_BIND_SERVICE `capability 授予某个进程，这样该进程就可以绑定 1-1023 的端口，而不必具有 root 权限。类似地，可以将 `CAP_SYS_ADMIN `capability 授予某个进程，这样该进程就可以执行系统管理任务，如挂载文件系统和创建设备节点等。

Linux 内核提供了一组默认的 capabilities，也可以通过自定义的方式创建新的 capabilities，以便更好地控制系统资源的访问权限。可以使用 `setcap `命令为二进制文件设置 capabilities。例如，下面的命令将 `CAP_NET_RAW `capability 授予 `/usr/bin/ping` 命令：

```bash
sudo setcap cap_net_raw+ep /usr/bin/ping
```

这样，用户就可以使用 `ping `命令而不必以 root 用户的身份登录。

除了 `CAP_NET_BIND_SERVICE `和 `CAP_SYS_ADMIN`，还有一些其他的 capabilities，以下是一些例子：

- `CAP_DAC_OVERRIDE`：允许进程忽略文件权限，可以访问任何文件。
- `CAP_CHOWN`：允许进程修改文件的所有者。
- `CAP_SETUID `和 `CAP_SETGID`：允许进程修改自己的用户 ID 和组 ID。
- `CAP_NET_ADMIN`：允许进程执行网络管理任务，如配置网络接口和路由表等。
- `CAP_SYS_RESOURCE`：允许进程修改系统资源限制，如 CPU 时间和内存限制等。

可以通过命令 `man 7 capabilities` 来查看系统提供的 capabilities 列表和详细说明。在使用 Kernel capabilities 时，需要注意，只有拥有 `CAP_SETFCAP `或 `CAP_SYS_ADMIN `capability 的进程才能够修改自己或其他进程的 capabilities，这也是为了保护系统的安全性。



如果执行 setcap 命令时出现 "command not found" 的错误，这通常意味着 setcap 命令所在的包尚未安装。在 openSUSE 中，setcap 命令包含在 libcap-progs 软件包中。

在 openSUSE 系统中需要安装 libcap-progs 软件包：

```bash
sudo zypper in libcap-progs
```

在 Ubuntu/Debian 系统上需要安装 libcap 库：

```bash
sudo apt-get install libcap2-bin
```

在 CentOS/RHEL 系统上需要安装 libcap 库：

```bash
sudo yum install libcap-devel
```

安装完成后，可以使用 setcap 命令为二进制文件设置 capabilities。如果还是无法找到 setcap 命令，可以尝试使用完整路径 /sbin/setcap 或者 /usr/sbin/setcap。



## seccomp策略

seccomp（secure computing mode）是 Linux 内核提供的一种安全机制，它可以限制进程能够进行的系统调用。通过使用 seccomp，可以限制进程只能够使用必要的系统调用，从而减少系统被攻击的风险。

seccomp 策略可以使用 BPF（Berkeley Packet Filter）语言编写，并使用 seccomp() 系统调用加载。以下是一个使用 seccomp 策略限制进程能够进行的系统调用的示例：

```c
#include <linux/seccomp.h>
#include <sys/prctl.h>
#include <unistd.h>

int main() {
    // 创建 seccomp 过滤器
    struct sock_filter filter[] = {
        BPF_STMT(BPF_LD | BPF_W | BPF_ABS, 0),
        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_write, 0, 1),
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL),
    };
    struct sock_fprog prog = {
        .len = sizeof(filter) / sizeof(filter[0]),
        .filter = filter,
    };

    // 加载 seccomp 过滤器
    if (prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog) < 0) {
        perror("prctl");
        return 1;
    }

    // 调用 write 系统调用
    char buf[] = "Hello, world!";
    write(1, buf, sizeof(buf));
    return 0;
}

```

上述代码创建了一个 seccomp 过滤器，仅允许进程调用 write() 系统调用，其他系统调用均会被禁止。可以通过编译并运行上述代码来演示 seccomp 策略的作用。

需要注意的是，seccomp 策略只能够限制进程进行的系统调用，但不能够限制系统调用的参数或返回值。因此，使用 seccomp 策略时需要特别小心，避免误用或产生漏洞。



## Netlink

Netlink 是一种 Linux 内核提供的通信机制，用于内核和用户空间进程之间的双向通信（IPC）。Netlink 可以用于许多目的，例如：

1. 配置网络设备和路由表：使用 Netlink 可以通过用户空间进程修改内核的网络设备和路由表配置，例如添加、删除、修改网络接口、IP 地址、路由等。

2. 监视网络事件：使用 Netlink 可以实时地从内核获取网络事件的通知，例如网络接口的状态变化、路由的变化等。

3. 程序间通信：使用 Netlink 可以在用户空间进程之间进行通信，类似于 Unix 域套接字。

Netlink 机制基于一种特殊的套接字类型（PF_NETLINK）和一个特定的协议（NETLINK）。用户空间进程可以通过创建 Netlink 套接字和内核通信。内核和用户空间进程之间的通信是基于 Netlink 消息的，每个 Netlink 消息包含一个消息头和一个负载（payload），负载可以是任何结构体或二进制数据。

Netlink 消息的类型和格式由内核定义。用户空间进程需要了解内核的 Netlink 消息格式和类型，才能正确地构造和解析 Netlink 消息。常用的 Netlink 消息类型包括：

1. RTM_NEWLINK 和 RTM_DELLINK：添加和删除网络接口。

2. RTM_NEWADDR 和 RTM_DELADDR：添加和删除 IP 地址。

3. RTM_NEWROUTE 和 RTM_DELROUTE：添加和删除路由。

4. RTM_NEWNEIGH 和 RTM_DELNEIGH：添加和删除 ARP 表项。

Netlink 可以使用 C 语言的 socket API 进行编程。

## Netfilter

Netfilter是Linux内核中的一个子系统，用于在数据包传输过程中进行过滤和操作。它支持对网络数据包进行各种类型的处理，包括过滤、修改、重定向等。Netfilter通过在内核中注册钩子函数，在数据包通过网络栈的不同阶段时进行拦截和处理。

Netfilter的核心是iptables命令，它可以用来配置Netfilter规则。iptables命令可以用来配置防火墙规则，NAT规则，限制连接速度等。iptables命令通过匹配不同的数据包字段（例如源IP地址、目的IP地址、源端口、目的端口等）来进行过滤。

除了iptables命令，还有其他一些工具可以用于配置Netfilter规则，例如nftables命令和firewalld服务。这些工具提供了更灵活、更强大的配置选项，可以帮助管理员更好地管理和保护网络安全。

也可以用于将网络数据包定向到单个容器。



更多信息可以参考 [LXC/LXD](https://linuxcontainers.org/)。



下面通过一个容器`alpine`的例子来演示在目录`/opt/test`下模拟实现根目录。

```bash
mkdir test
cd test
wget https://dl-cdn.alpinelinux.org/alpine/v3.13/releases/x86_64/alpine-minirootfs-3.13.4-x86_64.tar.gz
tar zxvf alpine-minirootfs-3.13.4-x86_64.tar.gz -C alpine-minirootfs/
```

查看当前目录结构：

```bash
tree ./test -L 1
```

输出结果：

```console
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

通过命令`unshare`挂载目录 `/opt/test/proc` 到某个文件来实现客户子系统。

```bash
sudo mount -t tmpfs tmpfs /opt/test/proc
```

```bash
sudo unshare --pid --mount-proc=$PWD/test/proc --fork chroot ./test/ /bin/sh
/ # ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    2 root      0:00 ps -ef
/ # touch 123
/ # ls 123
123
```

文件`123`在客户子系统中已创建，对应主系统中也可以对其进行读写操作。比如，修改文件`123`的内容。

```bash
su -
ls 123
echo hello > 123
```

文件`123`修改后的内容在客户机里面也可见。

```bash
/ # cat 123
hello
```

在主系统中再创建两个子目录 `/opt/test-1` 和`/opt/test-2`。

```bash
mkdir test-1
mkdir test-2
```

创建2个客户子系统，并将上面的两个子目录挂在到各自的 `/opt/test/home/`目录。

```bash
sudo mount --bind /opt/test-1 /opt/test/home/
sudo unshare --pid --mount-proc=$PWD/test/proc --fork chroot ./test/ /bin/sh
/ # cd /home
/home # echo "test-1" > 123.1
/home # cat 123.1
test-1
```

```bash
sudo mount --bind /opt/test-2 /opt/test/home/
sudo unshare --pid --mount-proc=$PWD/test/proc --fork chroot ./test/ /bin/sh
/ # cd /home
/home # echo "test-2" > 123.2
/home # cat 123.2
test-2
```

```bash
ll test/home
ll test-1/
ll test-2/
```

通过上面的演示，可以得出结论，两个客户子系统挂在到同一个主系统目录时，子系统时共享主系统目录，并相互影响。

## 安装Docker

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
