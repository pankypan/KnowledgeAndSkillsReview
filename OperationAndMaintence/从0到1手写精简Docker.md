# 从0到1手写精简Docker

## shell

### shell进程树

<img src="assets/005d23886a774c0fa7238eda5ca4b9fe.webp" width="500" />

这张图里的每一个红色的方框，都表示一个进程，并且箭头的指向代表进程之间的父子关系，它们形成了一个树状的结构。

具体说来，你的主机上运行着一个 **sshd 的守护进程**，每当一个用户通过 ssh 连接到主机时，这个 **sshd 守护进程**会创建一个 **sshd 子进程**，这个 **sshd 子进程**又会再创建一个 **shell 子进程**接收用户的指令。







### 熟悉而又陌生的 shell

那 shell 进程是什么呢？shell 就是用户与操作系统之间的接口，用于解释用户命令并执行相关的程序。



通过 `SHELL` 变量可以查看你当前会话的默认 shell 是什么，通过 `ps -p $$` 可以查看当前正在使用的 shell 是什么。

```bash
[shanke ~] # echo $SHELL
/bin/zsh
[shanke ~] # ps -p $$
PID TTY TIME CMD16643 pts/0 00:00:00 zsh
```

通过 `PS1` 变量可以查看并修改命令提示符前面的内容。

```bash
[shanke ~] # echo $PS1
[%n@%m %1~] %# 
[shanke ~] # PS1="[hehehehehehe]: "
[hehehehehehe]: PS1='[%m %1~] %#'
[shanke ~] #
```

**shell** 其实也是个普通的程序而已，只不过它**是系统启动后第一个和用户直接打交道的进程，并拥有直接执行其他进程的功能**，所以看起来比较特殊罢了。



看一下 shell 进程的源码。选择了比较简单的 xv6 源码中的 sh 实现，它精简地把所有 shell 程序都遵循的核心逻辑写出来了。

```c
int main(void) {
  static char buf[100];
  // 读取命令
  while(getcmd(buf, sizeof(buf)) >= 0){
    // 创建新进程
    if(fork() == 0)
      // 执行命令
      runcmd(parsecmd(buf));
    // 等待进程退出
    wait();
  }
}

void runcmd(struct cmd *cmd) {
  ...
  struct execcmd ecmd = (struct execcmd*)cmd;
  ...
  // 执行（替换为）用户传入的命令程序
  exec(ecmd->argv[0], ecmd->argv);
  ...
}
```

没错，shell 程序就是个死循环，它永远不会自己退出，除非我们手动终止了这个 shell 进程。

在死循环里面，就是持续不断地读取（getcmd）用户输入的命令（比如说 ls），创建一个新的进程（fork），在新进程里执行（exec）这个命令，最后等待（wait）进程退出，再次进入读取下一条命令的循环中。

这里的 `fork + exec` 是经典的 Linux 创建新进程并执行指定程序的方式。





**总结：**

用户分别在自己的 shell 里执行自己的命令，每执行一个命令就会创建一个新的子进程。那么进程树就会如下图所示。

<img src="assets/9f456b06a2fa5e0ef460ba960463dca6.webp" width="500" />

现在我们来总结一下：在你的主机上有一堆有着父子关系的进程们，其中有一个 **sshd 进程**监听着用户的远程登录。每当有用户连接过来时就创建一个 shell 子进程和用户交互，**shell 不断读取用户的命令并创建出新的进程。**







### 神秘的一号进程

既然 Linux 上所有运行着的东西其实就是一堆进程，那最开始是由谁来一步一步创造出这么多进程的呢？也就是说所有进程的初始发动机是什么？

```bash
[shanke ~] # ps -ef
UID   PID  PPID  C STIME TTY   TIME CMD
root    1     0  0 Jul17 ?     00:01:01 /usr/lib/systemd/systemd
```

这个 1 号进程就是 Linux 上的第一个进程，由最早的 Linux 版本中的 init 进程逐渐进化到现在的 systemd 进程，但本质上就还是第一个启动的一个普普通通的进程而已。

**systemd 进程**的细节比较多，但简单说其实就是**扫描几个指定目录下的特定后缀名的文件，然后解析出里面要执行的程序，把它执行起来。**Linux 上最初的一批由 systemd 启动的进程信息，都写在这些目录下。

比如刚刚我们看到的 sshd 进程，就写在了 `/lib/systemd/system/sshd.service` 这个文件中，可以看到这个文件中有一个 `ExecStart` 变量，后面就写着如何启动 sshd 服务。

```bash
[shanke ~] # cat /lib/systemd/system/sshd.service
[Unit]
Description=OpenSSH server daemon
Documentation=man:sshd(8) man:sshd_config(5)
After=network.target sshd-keygen.service
Wants=sshd-keygen.service

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/sshd
ExecStart=/usr/sbin/sshd -D $OPTIONS
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
```

所以说 Linux 上原本就只有 systemd 这一个进程，只不过这个进程的作用就是创造出超级多的子进程，这些子进程中功能比较复杂的可能又会创造出自己的子进程。

这样一来 Linux 上就充斥着各种各样的进程，整个系统就被 systemd 给盘活了。

<img src="assets/52a15968e714fe8e9e4b2044cbe00124.webp" width="600" />











## 根目录文件系统隔离(chroot)

### 如何隔离文件系统

在操作系统级别，文件系统是一个全局资源，被系统上运行的所有进程共享。

那我们现在的问题就很明确了，**如何让两个进程不去共享同一个文件系统，而是有一定隔离性呢**？



有一个简单的办法可以做到这一点，就是在主机的根目录中创建两个目录，分别给这两个用户使用，让他们以为自己的根目录就是你创建的这两个目录。

<img src="assets/6ff3b7e8ca133d71ae86ea94a560dd22.webp" width="500" />

那我们的问题此时更加明确了，就是如何让某个进程的根目录 / 变成我们指定的一个位置，而不是全部进程都共享同一个根目录呢？







### 大名鼎鼎的 chroot

这个事情非常容易，如果 Linux 内核提供这样的支持，即在每个进程的结构中记录了**根目录的位置**，而不是所有进程都使用同一个全局变量，那这个事情才有解。

<img src="assets/669c8b3dd9cc553082d6127a96a37f04.webp" width="500" />

Linux内核的chroot支持这个能力。在每个进程的结构体 task_struct 中用 root 字段记录了根目录的位置，这就做到不同进程可以不一样了。

```c
struct task_struct {
  ...
  struct m_inode * root;
  ...
};
```

为了修改这个字段的值，内核还提供了一个 chroot 的系统调用供上层用户调用，逻辑非常简单，就是把这个 root 字段修改为入参中传入的值。

```c
int sys_chroot(const char * filename) {
  struct m_inode * inode = namei(filename);
  ...
  current->root = inode;
  return (0);
}
```

画张图方便你理解。

<img src="assets/2c5e5c03c8e4830aaa189a54d3b6b888.webp" width="600" />

现在的 shell 和 glibc 库中也有对应的同名命令和库函数方便开发者调用，你可以分别 man 1 和 man 2 来查看详情。







### shell使用chroot

man 1 chroot 先看一下 chroot 命令的用法。

<img src="assets/0d3fd94c89218c2d32d0db1b0971dccb.webp" width="500" />

chroot 这条命令会执行你传入的 command 命令，同时改变进程的根目录。如果你没传具体的 command，就默认执行 $SHELL -i 命令，也就是启动一个新的 shell 进程。





**试验步骤：**

1. 创建dir，并使用chroot

   ```bash
   [shanke ~] # mkdir empty
   [shanke ~] # chroot empty 
   chroot: failed to run command ‘/bin/bash’: No such file or directory
   ```

2. Fix `\bin\bash`无法找到错误

   1. 错误原因

      <img src="assets/ecfa602ed23e63c3f635e97f51868cb6.webp" width="450" />

   2. 我们把主机上的 bash 拷贝过来，再次执行

      ```bash
      [shanke ~] # mkdir empty/bin
      [shanke ~] # cp /bin/bash ./empty/bin
      [shanke ~] # chroot empty 
      chroot: failed to run command ‘/bin/bash’: No such file or directory
      ```

3. Fix可执行文件(ELF)的动态链接问题

   1. 错误原因：找不到动态链接依赖

      <img src="assets/d0c7a193883b8eb1dd4bcc1768cddd09.webp" width="450" />

   2. 先执行下 file 命令看看文件的具体类型

      ```bash
      [shanke ~] # file /bin/bash
      /bin/bash: ELF 64-bit LSB executable ... dynamically linked ...
      ```

   3. 通过 ldd 命令可以查看到它所依赖的动态链接库

      ```bash
      [shanke ~] # ldd /bin/bash
        libtinfo.so.5 => /lib64/libtinfo.so.5 (0x00007fd34c3fc000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00007fd34c1f8000)
        libc.so.6 => /lib64/libc.so.6 (0x00007fd34be2a000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fd34c626000)
      ```

   4. 把这些 so 文件分别拷贝到我们的 empty 目录下同样的位置

      ```bash
      [shanke ~] # mkdir empty/lib64
      [shanke ~] # cp /lib64/libtinfo.so.5 empty/lib64
      [shanke ~] # cp /lib64/libdl.so.2 empty/lib64
      [shanke ~] # cp /lib64/libc.so.6 empty/lib64
      [shanke ~] # cp /lib64/ld-linux-x86-64.so.2 empty/lib64
      ```

4. 最终目录效果

   ```bash
   [shanke ~] # tree empty 
   empty
   ├── bin
   │   └── bash
   └── lib64
       ├── ld-linux-x86-64.so.2
       ├── libc.so.6
       ├── libdl.so.2
       └── libtinfo.so.5
   ```

5. 再次执行 chroot 命令，发现成功启动了一个新的 shell（命令提示符的前缀变成了 bash-4.2）！

   ```bash
   [shanke ~] # chroot empty                              
   bash-4.2# /bin/ls
   bash: ls: command not found
   ```







### C使用chroot

我们使用 C 语言来实现刚刚的效果，即创建一个新的 shell 进程，并更改它的根目录。

我们先实现一个最简单的版本，功能仅仅是运行后会创建一个新的 shell 进程，其他的什么都不做（这里我们不考虑任何的异常捕捉、内存释放等，就先写一个最 low 版本的代码，突出重点）

`demos/02-01-hello-shell.c`

```c
#include <stdlib.h>
#include <sys/wait.h>

int child();

// 直接运行 ./a.out
int main(int argc, char *argv[]) {
    // 创建新进程
    int pid = clone(child, malloc(4096) + 4096, SIGCHLD, NULL);
    // 等待子进程结束的信号
    waitpid(pid, NULL, 0);
    return 0;
}

int child() {
    char *args[] = {"sh", NULL};
    // 替换并执行指定的程序
    execvp("/bin/sh", args);
    return 1;
}
```

这段代码非常简单明了，就是用 clone 创建一个子进程，在子进程里通过 exec 替换为` /bin/sh` 程序，仅此而已。

直接编译执行一下，可以看到会弹出一个新的 **sh 进程** (`/bin/bash` 和 `/bin/sh` 是不同的，但通通都是 shell 的实现)

```bash
[shanke demos] # gcc 02-01-hello-shell.c
[shanke demos] # ./a.out 
sh-4.2# ps -f
UID    PID  PPID  C STIME TTY          TIME CMD
root  8733  8731  0 14:44 pts/0    00:00:00 bash
root  8826  8733  0 14:44 pts/0    00:00:00 ./a.out
root  8827  8826  0 14:44 pts/0    00:00:00 sh
root  9658  8827  0 14:47 pts/0    00:00:00 ps -f
```







我们在上面代码的基础上，再多实现一步，把进程的根目录给改掉。并把原来写死的 `/bin/sh` 改成通过命令行参数传入，这样我们可以灵活控制创建的新进程执行什么命令，或者使用哪一款具体的 shell。

`demos/02-02-chroot.c`

```c
#include <stdlib.h>
#include <sys/wait.h>

int child(void *argv);

// 直接运行 ./a.out empty /bin/bash
int main(int argc, char *argv[]) {
    // 创建新进程
    int pid = clone(child, malloc(4096) + 4096, SIGCHLD, argv);
    // 等待子进程结束的信号
    waitpid(pid, NULL, 0);
    return 0;
}

int child(void *arg) {
    char **argv = (char **)arg;
    // 设置根目录 这里 argv[1] 就是 empty
    chroot(argv[1]);
    // 把当前目录设置为根目录
    chdir("/");
    // 替换并执行指定的程序，这里 argv[2] 就是 /bin/bash
    execvp(argv[2], argv + 2);
    return 1;
}
```

`chdir()` 的意思是改变当前进程的工作目录。配合 chroot() 一块就是把根目录改了，再把当前目录指向根目录的意思。





实际上**命令行的 chroot** 内部就包含着把工作目录一块修改的逻辑，不然我们当时执行完 chroot 之后，为什么会很自然地直接跳转到新的根目录下呢。

通过 strace 来追踪 chroot 命令触发的系统调用可以看到这一点。

```bash
[shanke ~] # strace chroot empty 
...
chroot("empty") = 0
...
chdir("/")      = 0
...
```







## 特殊文件系统(mount)

在上一讲中，我们通过 chroot 和 chdir 在代码中实现了把一个 shell 进程的根目录修改为指定的位置。

<img src="assets/0e96b7e3787b0a1a255f11da9554e1d4.webp" width="500" />

但是留了一个问题，就是现在我们的 empty 目录太简陋了，而且只有 bash 和 ps 两个命令可以使用







### 麻雀虽小五脏俱全的 busybox

其实丰富这个目录，无非就是把需要的东西从主机上 copy 过来，但内容实在太多了。既然原理已经懂了，我们花那么多时间复制来复制去也没意思。

我们借用一个现成的迷你文件系统，busybox 镜像。这里面有几百个常用的 Linux 命令，并且都是静态编译，不需要依赖任何的动态链接库，所以可以无脑直接运行。这正是我们需要的！

```bash
docker run --name temp busybox:1.33
docker export temp -o busybox.tar
mkdir busybox
tar -xvf busybox.tar busybox/
```

搞好之后的目录结构是这样的。

```bash
busybox
├── bin
│   ├── ls
│   ├── top
│   ├── ps
│   ├── sh
│   ├── ...
├── dev
├── etc
│   ├── group
│   ├── localtime
│   ├── network
│   ├── passwd
│   └── shadow
├── home
├── proc
├── root
├── tmp
├── usr
│   └── sbin
└── var
    ├── spool
    └── www
```

重新运行一下我们的程序，这回根目录设置在 busybox 这里，看看效果。

```bash
[shanke ~] # ./a.out busybox /bin/sh
/ # PATH=/bin
/ # ls
bin   dev   etc   home  proc  root  tmp   usr   var
/ # pwd
/
/ # echo hello
hello
/ # hostname
shanke
```

趁热打铁，再多试几个命令看看。

```bash
/ # ps -ef
PID   USER     TIME  COMMAND
/ # top
top: no process info in /proc
```







### 神秘的 /proc 目录

 `ps` 和 `top` 命令是读取了`/proc` 目录里面的信息。

直接 strace 命令拦截下系统调用，看有没有打开（`open`）这个 `/proc`目录。

```bash
[shanke ~] # strace -e trace=open top
open("/proc/1/stat", O_RDONLY)      = 7
open("/proc/1/statm", O_RDONLY)     = 7
open("/proc/2/stat", O_RDONLY)      = 7
open("/proc/2/statm", O_RDONLY)     = 7

open("/proc/308/stat", O_RDONLY)    = 7
open("/proc/308/statm", O_RDONLY)   = 7

[shanke ~] # strace -e trace=open ps
open("/proc/1/status", O_RDONLY)    = 6
open("/proc/1/stat", O_RDONLY)      = 6
open("/proc/2/status", O_RDONLY)    = 6
open("/proc/2/stat", O_RDONLY)      = 6
...
open("/proc/308/status", O_RDONLY)  = 6
open("/proc/308/stat", O_RDONLY)    = 6
```

我省略了很多相似的内容，可以看到这俩命令都打开了大量的 /proc 目录。



这个 /proc 目录下给每个进程以它自己进程号为名称建立了一个子目录。而且主机上的进程是动态变化的，所以这个 /proc 目录下的内容不太可能存储到硬盘中。

通过`man proc`，查看`/proc`详细介绍。

![img](assets/39be7ba0ab81200362da44b0cfa5b9ce.webp)

首先 `/proc` 目录是反映进程信息的一个**伪文件系统**，也可以说是虚拟文件系统，它不是基于磁盘的传统文件存储，而是直接由操作系统内核或软件动态生成的文件和目录结构。

然后又告诉你，`/proc/[pid] `就表示给系统上所有跑着的进程创建的子目录，这些子目录下又存在很多格式如 `/proc/[pid]/XXX` 的目录。

`/proc/[pid]/cmdline`介绍：

![img](assets/c3acdeef806fbed18c21828c688a7ff8.webp)

没关系，直接拿我们最熟悉的 1 号进程（systemd）实践一下。

```bash
[shanke ~] # cat /proc/1/cmdline 
/usr/lib/systemd/systemd --switched-root --system
```

可以看到读取这个 cmdline 文件后，输出的内容是 1 号进程的启动命令，就是我们大名鼎鼎的 `systemd` 进程的**完整命令行**。





也就是说，本来你想要获取某进程的启动命令，需要调用内核提供的一个类似 `getSomeProcCmdline(pid)` 这样的方法，但现在不用了，直接和读取一个文件一样简单，就 `cat /proc/1/cmdline` 就行了。然后你以为是读一个文件，实际上内核里面调了个方法给你把数据返回回去了。

<img src="assets/fb3cf459adae0ad0d0604f3630b0952a.webp" width="600" />

所以伪文件系统理解起来很简单，它本质上就是内核以文件系统的接口（如 open, read, write）向外提供内部信息的一种机制。这种设计在某种程度上类似于 REST 风格的 HTTP 请求，通过标准化的方法访问资源，使得系统的内部状态和功能可以以一种直观且统一的方式进行交互和管理。







### 重看文件系统

我们对整个文件系统的理解又加深了，不但有从磁盘解析出来的一坨死数据那样的普通文件系统，还有像 /proc 目录这样由内核伪造出来的文件系统，它们都共同隐藏在根目录下

![img](assets/781ff728ba8eb0758c636c7ee77dcf45.webp)

不论是**普通文件系统**还是**伪文件系统**，都有自己对应的**类型（Type）**，比如根目录 / 的类型是 ext4，/proc 的类型是 proc，/sys 的类型是 sysfs 等。可以使用 **df 命令**来查看一个目录所在的文件系统的信息。

```bash
[shanke ~] # df -hT /
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/vda1      ext4   99G  4.4G   90G   5% /
[shanke ~] # df -hT /proc
Filesystem     Type  Size  Used Avail Use% Mounted on
proc           proc     0     0     0    - /proc
[shanke ~] # df -hT /sys 
Filesystem     Type   Size  Used Avail Use% Mounted on
sysfs          sysfs     0     0     0    - /sys
```

拿普通文件系统 / 来说，查出的信息表示它的内容是从 /dev/vda1 这个设备加载的（这是我云主机上的虚拟硬盘，自己的电脑可能是实际的物理硬盘 /dev/sda1），获取的数据按照 ext4 这种文件系统格式解析，然后在操作系统中以目录树的形式展现。







### 任性的挂载

在根目录下创建一个新的目录叫 hehe，然后给 hehe 挂载一个 proc 文件系统。

```bash
[shanke /] # mkdir hehe
[shanke /] # mount -t proc proc /hehe
```

<img src="assets/4f0d8e5fac03f4738e0bee7e2d61c9d6.webp" width="600" />





我们再做件更好玩的事儿，给 hehe 目录挂载一个和我现在根目录 / 一模一样的文件系统，也同样是从硬盘设备 /dev/vda1 读取内容并按 ext4 格式解析。

PS：记得先卸载之前给 hehe 挂载的 proc 文件系统。

```bash
[shanke /] # umount /hehe
[shanke /] # mount -t ext4 /dev/vda1 /hehe
```

<img src="assets/3195708a15580d733e30088fd4fc5796.webp" width="600" />





`man mount` 一下，简单补充下理论知识。

![img](assets/b7a6d7a39d8e040a2f37e8f3c9640e00.webp)









### shell使用mount

讲了这么久，其实总结起来就一句话，/proc 不是个普通的目录，它是直接挂载了一个 proc 文件系统形成的伪文件系统。

所以你直接 copy 主机的 /proc 目录是没用的，会发生很多奇奇怪怪的事儿。

那接下来就简单了，我们回到刚刚我们执行 ps 不对的地方，挂载一下。

```bash
[shanke ~] # ./a.out busybox /bin/sh
/ # PATH=/bin
/ # ps -ef
PID   USER     TIME  COMMAND
/ # mount -t proc proc /proc
/ # ps -ef
PID   USER      TIME  COMMAND
    1 root      1:19 /usr/lib/systemd/systemd
...
18410 root      0:00 ./a.out busybox /bin/sh
18411 root      0:00 /bin/sh
18413 root      0:00 ps -ef
```







### C使用mount

`man 2 mount` 看下是否有现成的库函数。

![img](assets/99dee5fce164bcc8292fe248eb46188f.webp)





`demos/03-01-mount.c`

```c
#include <stdlib.h>
#include <sys/wait.h>
#include <errno.h>
#include <sys/mount.h>

int child(void *argv);

// 直接运行 ./a.out empty /bin/bash
int main(int argc, char *argv[]) {
    // 创建新进程
    int pid = clone(child, malloc(4096) + 4096, SIGCHLD, argv);
    // 等待子进程结束的信号
    waitpid(pid, NULL, 0);
    return 0;
}

int child(void *arg) {
    char **argv = (char **)arg;
    // 设置根目录
    chroot(argv[1]);
    // 把当前目录设置为根目录
    chdir("/");
    // 挂载 proc 文件系统
    mount("proc", "/proc", "proc", 0, "");
    // 替换并执行指定的程序
    execvp(argv[2], argv + 2);
    perror(argv[2]);
    return 1;
}
```

运行下我们的代码

```bash
[shanke ~] # gcc demos/03-01-mount.c -o skdocker
[shanke ~] # ./skdocker busybox /bin/sh
/ # PATH=/bin
/ # ps -ef
PID   USER      TIME  COMMAND
    1 root      1:19 /usr/lib/systemd/systemd
...
18410 root      0:00 ./skdocker busybox /bin/sh
18411 root      0:00 /bin/sh
18413 root      0:00 ps -ef
```







### 突然感觉哪里不太对劲

说白了就是，我们最初的办法是，通过搞两个不同的目录，并且把两个用户的进程分别 chroot 到这俩目录里，形成各自的小空间，彼此独立，互相不影响。

但此时这个 /proc 目录似乎没有被关在这个小空间里。

给这两个用户分别启动两个 shell 进程，一个 chroot 到 busybox1 ，另一个 chroot 到 busybox2，双方都挂载了自己根目录下的 proc。

此时他们各自通过 ps 命令是可以看到对方的进程的。

```bash
[shanke ~] # ./skdocker busybox1 /bin/sh
/ # PATH=/bin
/ # ps -ef
PID   USER      TIME  COMMAND
    1 root      1:19 /usr/lib/systemd/systemd
...
18410 root      0:00 ./skdocker busybox1 /bin/sh
18411 root      0:00 /bin/sh
18413 root      0:00 ps -ef
```

```bash
[shanke ~] # ./skdocker busybox2 /bin/sh
/ # PATH=/bin
/ # ps -ef
PID   USER      TIME  COMMAND
    1 root      1:19 /usr/lib/systemd/systemd
...
18410 root      0:00 ./skdocker busybox1 /bin/sh
18411 root      0:00 /bin/sh
18420 root      0:00 ./skdocker busybox2 /bin/sh
18421 root      0:00 /bin/sh
18429 root      0:00 ps -ef
```









## 名称空间(namespace)

上一讲我们通过挂载把 `/proc` 目录搞好了，但有个问题就是`/proc` 目录没有做到隔离。你的两个用户分别查看这个目录里的内容，比如 `ps` 命令，是可以看到另一个用户创建的进程信息的。

怎么解决这个问题呢？也就是说如何让一个进程在执行 `ps` 命令的时候，只看到一定范围的进程信息呢？

![img](assets/d99be34dca8b776e394df38ccb06a86c.webp)





Linux内核的**namespace**是一种特性，用于实现进程间的资源隔离。它允许在不同的namespace中有相同的资源名称，从而使得每个进程或进程组可以看到自己的独立视图。

主要功能包括网络、进程ID、挂载点等的隔离，常用于容器技术，使得多个容器可以在同一主机上安全地运行而互不干扰。这种机制的ultimate目的是提高安全性和资源管理效率。





### 有需求就加个字段

在进程的结构体上加个字段，**通过这个字段的值给每个进程打个标签，然后每个进程只能看到打了同一个标签的进程，就好了。**

这个标签就在每个进程的结构体 `task_struct` 里，最终指向有个类型叫 `pid_namespace` 的字段。

在内核中，就具体体现为每个进程结构中的 `nsproxy` 字段，其中对 `pid`（相当于一个标识符）进行隔离的就是 `pid_namespace`，对 `uts` 进行隔离的就是 `uts_namespace`。

![img](assets/5ebde60052f9f1202f72c181ba5bf779.webp)





下面列出在 Linux 3.10 版本下存在的，也是 docker 所使用的所有的 namespace，以及对应在调用 clone 方法时传递的 flags 标识。

![img](assets/c676c415b29742c35c6c840a90d15c31.webp)









### 如何修改命名空间

我们现在改命名空间，一开始所有进程的命名空间如果不特殊指定，也是全都指向一个默认的命名空间（`init_nsproxy`），那必然有一个系统调用可以帮我们修改。

<img src="assets/8b22346b130b55998c6176cd7d4d8408.webp" width="600" />

有三个系统调用都可以修改这个字段，分别是：

- `clone()`：创建一个新的进程，通过传入一个标志位来决定是否指向新的命名空间。
- `unshare()`：使某进程脱离当前的命名空间。
- `setns()`：把某进程加入到某个命名空间。



用图说话就是这样。

<img src="assets/d5a29dffbd2795084010051f9553fa42.webp" width="600" />







### shell完成UTS 命名空间的隔离

使用 shell 自带的命令 `unshare` 来实现

<img src="assets/18fa97e590bd73ab62eeaa2aaa7a9f0d.webp" width="600" />

<img src="assets/c5cc77179cbecae98fe100864873f6e1.webp" width="600" />





**实践案例：**

```bash
// 最外层 shell 的主机名是 shanke
[shanke ~] # hostname
shanke
[shanke ~] # unshare -u bash
// unshare -u 的 shell 里把主机名改成 hehehehe
[bash] # hostname hehehehe
[bash] # hostname
hehehehe
[bash] # bash
// 再创建子进程 shell 发现主机名被修改为 hehehehe
[bash-bash] # hostname
hehehehe
// 此时把主机名改为 HAHAHAHA
[bash-bash] # hostname HAHAHAHA
[bash-bash] # exit
// 退出发现父 shell 进程的主机名被改成 HAHAHAHA
[bash] # hostname
HAHAHAHA
[bash] # exit
// 但最初的那个最外层 shell 主机名依然是最开始的 shanke
[shanke ~] # hostname
shanke
```

总结起来就是 unshare 创建出的父子进程的主机名互相独立，直接创建出的父子进程主机名共享。在刚刚的图中继续补充完整就是下面这样。

<img src="assets/dce3811f66c5230571863789cf3bc868.webp" width="600" />





要想让你的两个用户对这些更深层次的系统资源（比如主机名、PID 等）有一定的隔离性，那只需要分别 unshare 出一个 shell 进程给他们用就好了，每个人都在自己的命名空间下折腾，互不干扰。

<img src="assets/82569af2979fe6a45b9ee2990a7a2eb7.webp" width="600" />





### C完成UTS命名空间的隔离

之前我们在代码里创建新进程，用的是 clone 方法，这个 clone 是 glibc 的函数库。

```c
clone(child, malloc(4096) + 4096, SIGCHLD, argv);
```

clone 方法有四个参数：

1. 表示子进程从哪个方法开始，这里是 child 方法。
2. 表示子进程的堆栈指针，这里直接申请了一个 4K 大小的栈。
3. 表示进程的 `flags` 标识，是一组位掩码，用于指定新进程的行为和从父进程继承哪些资源。
   1. 这里的 `SIGCHLD` 表示子进程终止时会向父进程发送一个 SIGCHLD 信号。
4. 表示传递给子进程方法（`child`）的参数。





我们如果想让子进程独立出 `uts 命名空间`，那么只需要在 `clone` 的 `flags` 参数里多加一个 `CLONE_NEWUTS` 就可以了！

源代码在 `demos/04-01-uts.c` 中

```c
int pid = clone(child, malloc(4096) + 4096, CLONE_NEWUTS | SIGCHLD, argv);
```

本次代码（容器进程修改 hostname 没有影响了其他进程）

```bash
[shanke ~] # hostname
shanke
[shanke ~] # gcc demos/04-01-uts.c -o skdocker
[shanke ~] # ./skdocker busybox /bin/sh
/ # hostname hehehehe
/ # exit
[shanke ~] # hostname
shanke
```







### C完成全部命名空间的隔离

源代码在 `demos/04-02-allns.c` 中

```c
// 创建新进程的命名空间标识
int flags = CLONE_NEWUTS | CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET | CLONE_NEWIPC;
// 创建新进程，新进程在上述独立的命名空间下
int pid = clone(child, malloc(4096) + 4096, flags | SIGCHLD, argv);
```

我们直接运行下代码，看看之前说的 ps 命令能看到主机上所有进程的问题有没有解决。

```bash
[shanke ~] # gcc demos/04-02-allns.c -o skdocker
[shanke ~] # ./skdocker busybox /bin/sh
/ # ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    2 root      0:00 ps -ef
```

快速验证下其他的命名空间是否隔离

`验证ipc命名空间`

```bash
/ # ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
```

`验证 mount 命名空间`

```bash
/ # mount
proc on /proc type proc (rw,relatime)
```

`验证 net 命名空间`

```bash
/ # ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```





**mount命名空间的shared subtree机制的影响：**

不过，这里有个稍微复杂的小问题，就在这个 **mount 命名空间**里。我们预期的结果应该是处在独立的 mount 命名空间下的容器，它的挂载（mount）和卸载（umount）动作对宿主机是不影响的。但是由于 **mount 命名空间**里比较特殊的 `shared subtree` 机制，比如如果一个挂载点被标记为 "shared"，则该挂载点上的挂载和卸载事件会传播到与其 "shared" 关系的所有挂载点。

所以其实我们的容器中的挂载和卸载动作是会被宿主机看到的，最严重的情况就是如果你的容器根目录和宿主机根目录一样，那么你容器中卸载 `/proc` 目录，将会导致宿主机的 `/proc` 目录失效。你可以稍稍改一下最后一版代码来尝试下这个现象。

这也是很多书中和资料中犯的错误，认为独立了 mnt 和 pid 命名空间并且在自己实现的容器中重新挂载 /proc 目录，看到只有自己的进程，就证明满足了隔离性，这其实是有问题的。

解决办法也很简单，就是在进入容器后立刻把所有挂载点都设为 "private"，这样就做到我们所预期的那种完全隔离了。这块就不展开讲解了，优化的代码在 `demos/04-03-update-mnt.c `中，运行这个会安全地挂载` /proc` 目录而不会影响宿主机，在 `docker` 的源码中也是这样做的。







### 可以说出容器这个词了

你可能会发现，在文中我慢慢把启动一个进程这个说法，换成了启动一个容器。

没错，到现在这一步，虽然我们**本质上仍然只是启动了一个子进程，但这个子进程由于被我们做了很多的隔离**，慢慢变得有些与众不同了。

而且这个隔离的子进程，由它所再创造的**子进程**们组成的**一棵进程树**，都会继承这些隔离的性质。那么这些进程共同被隔离的样子，就很像被一个真实存在的**容器**包裹起来一样。

由这些子进程们执行的操作，以及创建新的子进程的过程，我们就可以形象地称之为「在容器中」进行的什么什么动作。











## 资源控制(cgroup)

### 让用户郁闷的 top 命令

我们来做个实验，在宿主机上把 cpu 打满。

你可以用 C 或者 shell 程序写个死循环来跑，不过这里我们用一个更方便的工具 stress。

在一个 4 核的宿主机上执行下 `stress -c 4` 命令，这表示启动 4 个进程分别执行一个非常耗 CPU 的程序，意思就是把 4 个 CPU 跑满。

![img](assets/149b8f9b4821d63e35449ee85ffaba04.webp)

可以看到，用 top 命令查看，每个 `stress` 进程都占有几乎 100% 的 CPU 运行时间，因为每个 CPU 都被一个 stress 进程耗满了。





接下来我们换个方式。分别启动两个容器（左边是容器 1，右边是容器 2），在每个容器里执行 `stress -c 4` 命令，再用 `top` 命令分别查看。

![img](assets/71e2ee623c9779fbd57f823068dabc5e.webp)

可以看到此时每个 `stress` 进程都只能占了 50% 的 CPU 时间。

站在上帝视角的你，也就是宿主机层面看，当然能理解。因为实际上一共机器上启动了 8 个 stress 进程，那一共 4 个 CPU 当然只能给每个 stress 进程分配 50% 左右的时间。

但是处在容器中的两个用户就会很郁闷，怎么自己买的服务器 CPU 跑不满呢？然后也找不到其他吃 CPU 的进程是哪个（因为进程信息被之前的 `pid` 命名空间隔离了）







### 控制组 cgroup

`cgroup`（控制组）， 就是 Linux Control Group，是 Linux 内核的一个特性，用于限制、记录和隔离进程组的资源使用（如 `CPU、内存、磁盘 I/O` 等）。通过 `cgroup`，系统管理员可以精确控制分配给每个进程组的资源量，并且可以嵌套使用。

作为使用者，使用 `cgroup` 功能是非常简单的。`cgroup` 和 `proc` 一样，也被 Linux 设计成了一个虚拟文件系统，通过文件的读写来控制和查看 `cgroup` 相关的功能。

整个操作非常简单直观，就是先创建一个目录代表一个分组，然后把一个进程号写入 tasks 文件表示这个进程被这个分组控制，最后把一些值写入某些文件表示控制 XXX 资源的使用限制。太符合直觉了。

可以看出一旦一个内核功能被设计成了虚拟文件系统的形式，不论其原理有多复杂，对使用者来说操作起来都是非常丝滑无脑的，类似傻瓜式设计。当然缺点就是得习惯一下，不能真把它当普通的文件来理解。

`cgroup` 是个大的虚拟文件系统，具体还分为很多子系统，比如刚刚管理 `cpu` 的就是 `cpu 子系统`，管理内存的就是`内存子系统`。

`lssubsys` 可以看到所有的子系统列表。

```bash
[shanke ~] # lssubsys -am
cpuset /sys/fs/cgroup/cpuset
cpu,cpuacct /sys/fs/cgroup/cpu,cpuacct
memory /sys/fs/cgroup/memory
devices /sys/fs/cgroup/devices
freezer /sys/fs/cgroup/freezer
net_cls,net_prio /sys/fs/cgroup/net_cls,net_prio
blkio /sys/fs/cgroup/blkio
perf_event /sys/fs/cgroup/perf_event
hugetlb /sys/fs/cgroup/hugetlb
pids /sys/fs/cgroup/pids
```

<img src="assets/80c4477ff91b21960e6a51472b14b540.webp" width="500" />









### shell完成cgroup控制

**控制CPU使用：**

- 首先在 `/sys/fs/cgroup/cpu` 目录下，创建一个新的目录 `skdocker`。

  ```bash
  mkdir /sys/fs/cgroup/cpu/skdocker
  ```

- 我们把第一个 stress 进程的 PID 4620 写入 tasks 这个文件里。

  ```bash
  # 表示这个进程受本组 cgroup 的管控
  echo 4620 > /sys/fs/cgroup/cpu/skdocker/tasks
  ```

- 然后再把一个数字 50000 写入 cpu.cfs_quota_us 这个文件里。

  ```bash
  # 表示这个 cgroup 管控的进程总共只能使用 50% 的 CPU 资源
  echo 50000 > /sys/fs/cgroup/cpu/skdocker/cpu.cfs_quota_us
  ```





**控制内存使用：**

```bash
[shanke ~] # mkdir /sys/fs/cgroup/memory/skdocker
[shanke ~] # echo 2g > /sys/fs/cgroup/memory/skdocker/memory.limit_in_bytes
[shanke ~] # echo 24241 > /sys/fs/cgroup/memory/skdocker/tasks
```









### C完成cgroup控制

每个容器 CPU 使用限制 `20%`，我们用一个比较新颖的，也更方便的方法，直接把 shell 命令写在代码里，和我们刚刚的操作一模一样。

完整代码在 `demos/05-01-cgroup-cpu.c`

```c
...

int main(int argc, char *argv[]) {
    int flags = CLONE_NEWUTS | CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET | CLONE_NEWIPC;
    int pid = clone(child, malloc(4096) + 4096, flags | SIGCHLD, argv);
    // 设置 cgroup
    cfcgroup(pid);
    waitpid(pid, NULL, 0);
    return 0;
}

int child(void *arg) {
    ...
}

void cfcgroup(pid_t pid) {
    system("mkdir -p /sys/fs/cgroup/cpu/skdocker");
    systemf("mkdir -p /sys/fs/cgroup/cpu/skdocker/%d", pid);
    systemf("echo %d >> /sys/fs/cgroup/cpu/skdocker/%d/tasks", pid, pid);
    systemf("echo 20000 > /sys/fs/cgroup/cpu/skdocker/%d/cpu.cfs_quota_us", pid);
}

void systemf(const char *format, ...) {
    char cmd[128];
    va_list args;
    va_start(args, format);
    vsnprintf(cmd, sizeof(cmd), format, args);
    va_end(args);
    system(cmd);
}
```

看，就是简单把刚刚的 shell 命令写在代码里了而已，这里使用了 system() 方法，直接调用了 shell 命令。同时，我们把每个容器的 cgroup 分组以这个容器进程的 pid 也就是容器里的 1 号进程为目录名。

<img src="assets/75b93f7e71c9ef0b65eed58ccffd677c.webp" width="600" />

编译、运行，在启动的容器里运行 stress 把 CPU 占满，这时在宿主机 top 一下就发现只能占 20% 左右了！

![img](assets/5678a4e9d5fd83a948198580824014d6.webp)

查看一下这四个 stress 进程的 cgroup 分组，发现都被放在了 11333 这个目录下。

```bash
[shanke ~] # cat /proc/11471/cgroup | grep cpuacct,cpu
3:cpuacct,cpu:/skdocker/11333
[shanke ~] # cat /proc/11472/cgroup | grep cpuacct,cpu
3:cpuacct,cpu:/skdocker/11333
[shanke ~] # cat /proc/11473/cgroup | grep cpuacct,cpu
3:cpuacct,cpu:/skdocker/11333
[shanke ~] # cat /proc/11474/cgroup | grep cpuacct,cpu
3:cpuacct,cpu:/skdocker/11333
```

而 11333 这个进程刚好是容器里的 1 号进程 /bin/bash！非常符合预期。

```bash
[shanke ~] # ps -ef|grep /bin/bash
root     11331 30321  0 00:02 pts/1    00:00:00 ./skdocker ../centos7 /bin/bash
root     11333 11331  0 00:02 pts/1    00:00:00 /bin/bash
```











## 容器网络(birdge)

### 什么叫无法上网

怎么创建一个容器之后网络就成了一个需要考虑的问题了呢？这或许才是首先会产生困惑的问题。

别废话，直接看现象。

- 启动一个容器

  ```bash
  [shanke ~] # ./skdocker busybox /bin/sh
  ```

- 先 `ping` 一下百度，失败。

  ```bash
  / # ping www.baidu.com
  ping: bad address 'www.baidu.com'
  ```

- 再 `ping` 一个已知的 `ip`，失败。

  ```bash
  / # ping 110.242.68.3
  PING 110.242.68.3 (110.242.68.3): 56 data bytes
  ping: sendto: Network is unreachable
  ```

- 甚至 `ping` 本地回环地址，还是失败。

  ```bash
  / # ping 127.0.0.1
  PING 127.0.0.1 (127.0.0.1): 56 data bytes
  ping: sendto: Network is unreachable
  ```

很简单，这些就是无法上网，大白话就是网络相关的通信哪哪都不通。



宿主机如何上网：

1. 宿主机能上网并不是自然而然的事情，而是背后做了一些网络的配置工作。
2. 容器的网络通过 net 命名空间与宿主机隔离，一个干净的初始状态的网络环境，是几乎啥功能都没有的。









### 解决 ping 127.0.0.1 的问题

**网络接口**表示系统中所有可以进行**网络通信的组件**，包括

- 物理硬件设备（比如这里的 eth0 以太网卡，eth 表示 ethernet，翻译过来就是以太网；以及可以连 WIFI 的无线网卡）
- 虚拟设备（比如 veth、VPN、bridge 等，上图中还没有这类设备）
- **逻辑接口**（比如这里的 lo 回环接口，就是我们常说的 localhost，数据不通过网络传出机器，而是在本地直接路由给自己）。






首先用 `ip addr` 命令查看一下系统中的所有网络接口信息。

<img src="assets/5b5aa981cd5931af162a436d9a40124e.webp" width="600" />

诶？那既然容器中也有 lo 回环接口，为什么 `ping 127.0.0.1`还不行呢？

答案简单的很，因为没开启。直接通过命令 `ip link set lo up` 开启一下，瞬间就好了。

<img src="assets/84658afae6da80446f7d15584b8edcce.webp" width="600" />



可以用 `tcpdump` 命令抓取一下流经 lo 回环接口的数据包，查看数据流向。







### 解决 ping 外部网络的问题

接下来想要解决访问外部网络的问题，就必须得有个实实在在的物理网卡可以让我们把数据发出去，经过真实的网线（或者 WiFi）传播向远方。

必须有一种虚拟化的手段，即便主机上只插了一块物理网卡，但可以搞出多个**虚拟网卡**，让每个容器都以为自己占用了一个独立的物理网卡一样。

<img src="assets/890e08aa7e90cec4f464469e04a41c95.webp" width="600" />

容器里的网络数据都发给自己的这个虚拟的 eth0，在容器层面就以为是从真实物理网卡发出去了。

然后在宿主机层面，通过软件的方式把每个容器的 eth0 收到的数据包，真真正正通过插在主机上的网卡发出去，整个过程就结束了。





上面只是我们的构想，具体还得看 Linux 内核老大哥有没有提供支持。

目前主流的**虚拟网卡方案**有 `tun/tap` 和 `veth` 两种，容器一般是使用 `veth` 来实现网络通信的，所以我们也使用 `veth` 来实现我们的目标。

过程有一丢丢复杂，我先把宿主机和容器中需要的命令写出来，从行数看就没多少步骤了。

```bash
--- 宿主机 ---
# 添加 veth 设备，一端在宿主机（veth-host），另一端在容器（veth-container）
ip link add veth-host type veth peer name veth-container
ip link set veth-container netns 容器PID
# 给宿主机一端的 veth-host 配置 IP 并开启
ip addr add 172.16.0.1/16 dev veth-host
ip link set veth-host up
# 开启 nat 转换
iptables -t nat -A POSTROUTING -s 172.16.0.2/16 -o eth0 -j MASQUERADE
sysctl -w net.ipv4.ip_forward=1

--- 容器 ---
# 给容器一端的 veth-container 配置 IP 并开启
ip addr add 172.16.0.2/16 dev veth-container
ip link set veth-container up
# 配置默认网关
ip route add default via 172.16.0.1
```

**关键点解释：**

1. 首先在宿主机上添加一对儿 `veth` 设备，一端自然就连在宿主机上（相当于在宿主机的网络命名空间里），另一端放进容器的网络命名空间里。

   <img src="assets/0476685033889cb6594cfff976775115.webp" width="400" />

2. 接下来分别给这两个设备配置 **IP 地址和子网掩码**，让 `veth-host` 和 `veth-container` 处于同一个子网内，但不要和 eth0 处于同一个子网内。

   <img src="assets/1cada2a143177b166c68da2776e17a8f.webp" width="400" />

3. 上一步完成后，宿主机可以 ping 通容器，容器也能 ping 通宿主机了！

   1. 但此时容器还不能 ping 通宿主机上的 eth0 这块真实的物理网卡，更不要说 ping 通外网了。

      ```bash
      / # ping 10.24.0.5
      PING 10.24.0.5 (10.24.0.5): 56 data bytes
      ping: sendto: Network is unreachable
      ```

   2. 原因是目前容器中的路由表信息，只能路由到 172.16.0.0/16 这个子网内的 IP。

      ```bash
      / # ip route
      172.16.0.0/16 dev veth-container scope link  src 172.16.0.2 
      ```

4. 给容器添配置一个默认网关。这样在容器便能 ping 通宿主机上的 eth0 这块真实的物理网卡。

5. 现在还差最后一步，就是访问外部的网络还是不行，

   1. 不过不是网络不可达（因为我们已经配置了默认网关，只要不知道该发到哪的通通都会发到默认网关，一定能发出去），而是收不到响应。
   2. 这是因为你使用的 `veth-container` 这个网卡的地址是 `172.16.0.2`，这是个**内网地址**。往出发包可以找到目的 IP，但是响应回来的时候公网上可找不到你这个内网地址。所以这里我们需要进行 **NAT 转换技术**，简单说就是**先把你的内网 IP 转变成网卡的公网 IP，发出去，等响应数据包回来时，再把公网 IP 转变成内网 IP，在主机内流转到你的容器中的虚拟网卡上**。
   3. 因此，**开启NAT转换**，解决响应问题。







### 解决容器间互 ping 的问题(bridge)

在我们的虚拟世界里，容器和宿主机的 veth 设备，也可以连接到一个共同的虚拟设备上，实现多方互连。

这个设备就叫做 bridge，中文名叫网桥，但实际上它使用起来就像是一个虚拟的交换机。





**bridge 模式**

<img src="assets/cc0c3dd4d607f53d2f0e928efe205dde.webp" width="500" />

那实现这个图的效果，只需要简单几行命令即可。

```bash
# veth-host 不需要 ip 地址了删掉它
ip addr delete 172.16.0.1/16 dev veth-host
# 建立网桥设备 br0
ip link add name br0 type bridge
# 把 veth-host 插入网桥
ip link set veth-host master br0
# 给网桥设置 ip 地址
ip addr add 172.16.0.1/16 dev br0
# 最后别忘了开启它
ip link set br0 up
```

这里我们把 `veth-host` 的 `ip` 地址去掉了，给了网桥设备` br0`，相当于 `veth` 的一端不再需要一个设备来承载，直接变成一根啥也不是的网线插在了 `br0` 上，剩下的一切由 `br0` 去承担。

接下来我们通过设置 `veth-host` 的 `master` 为 `br0`，实现了插在 `br0` 上这个动作，也叫做**桥接**。







### C代码实现网络功能

由于所有的容器都依赖宿主机上的网桥设备，所以需要先在宿主机上初始化网桥，代码在 `init_net.sh` 中，很简单。

```shell
# 建立网桥设备 br0
ip link add name br0 type bridge
# 给网桥设置 ip 地址
ip addr add 172.16.0.1/16 dev br0
# 最后别忘了开启它
ip link set br0 up
```

然后修改容器代码实现，在 demos/06-02-net-bridge.c 中，稍稍复杂了点。这里我只把要点列出，感兴趣期望你去代码仓库看看完整版，有不少细节。

```c
// 直接运行 ./a.out empty /bin/bash
int main(int argc, char *argv[]) {
    ...
    // 设置网络
    cfnet(pid);
    ...
    }

int child(void *arg) {
    // 设置容器的网络
    child_cfnet();
    ...
}

void cfnet(pid_t container_pid) {
    systemf("ip link add veth-host-%d type veth peer name veth-container", container_pid);
    systemf("ip link set veth-container netns %d", container_pid);
    systemf("ip link set veth-host-%d up", container_pid);
    systemf("ip link set veth-host-%d master br0", container_pid);
    printf("父进程设置网络完毕，设备为 veth-host-%d\n", container_pid);
}

void child_cfnet() {
    srand(time(NULL));
    int random_num = rand() % (254 - 2 + 1) + 2;
    system("ip link set lo up");
    system("ip link set veth-container up");
    systemf("ip addr add 172.16.0.%d/16 dev veth-container", random_num);
    system("ip route add default via 172.16.0.1");
    printf("echo 容器设置网络完毕，设备为 veth-container:172.16.0.%d/16\n", random_num);
}
```

编译运行一下，可以发现我们最新版的容器，可以访问百度，也可以访问同宿主机上的其他容器了。

![img](assets/1317eb70b100d7136b26441caf63fc16.webp)

每个容器有自己独立的 IP 地址，可以互联互通和访问外网，所有网络配置的变更也不会影响其他容器。这可真是个不小的成就呢！







### docker 的网络

docker 在启动一个容器时，可以通过 --network 指定网络驱动，也就是选择网络配置的方式。

- --network=none 表示隔离网络命名空间但不进行任何网络配置，和我们一开始没进行任何网络配置时实现的容器代码一样。
- --network=host 表示不隔离网络命名空间，即我们一开始的使用宿主机网络实现上网的原理一样。
- --network=bridge 表示网桥模式，和我们最后通过 veth 虚拟网卡 + bridge 网桥实现的网络一样，也是 **docker 默认的网络配置**。

<img src="assets/9c74624e58ea7aba7809db7c7d878271.webp" width="600" />





那么通过 docker 启动一个容器后（默认使用 bridge 驱动），分别查看宿主机和容器的网络配置，你就不再陌生了。

<img src="assets/eb40849733fe113dd56af802c431af8b.webp" width="600" />

- 可以清晰地看到宿主机上有个网桥叫 docker0，这是在你启动 docker 时就创建好的（systemctl start docker），和我们初始化宿主机网络的脚本类似。
- 然后容器和宿主机上有一对儿 veth 设备，容器端叫 eth0@if81，宿主机端叫 vetha562d7a@if80，同时宿主机端的 veth 插到了 docker0 上。

这就是 `veth + bridge` 技术的**容器网络实现方案**，和我们实现的完全一样！









## 容器和镜像分家(unionfs)

### 两个相同根路径的容器相互影响

现在我们要运行两个根目录一样的容器。

<img src="assets/445ad25ffa89398af70c1da21286e9ee.webp" width="600">

然后上面的窗口添加点内容。

<img src="assets/fe5ab013e22e177510b37134b9ec536e.webp" width="600">

我们发现两个容器连文件系统都没隔离开。

原因很好理解，我们可以通过 `chroot` 和 `chdir` 把两个容器的文件系统隔离开，**前提是弄到两个不同的目录上**。现在弄到同一个目录下一起改，肯定就互相影响了。





### 单纯的复制

很简单，每次启动一个容器的时候，都把我们的传入的目录复制一份新的，以这个新的目录为容器根目录。

- 我们把最原始的目录叫做**镜像（image）**
- 把**创建一个容器时复制出来的这个目录**叫做**容器的根目录**，有的时候也可以简称为**容器（container）**。

在某些语境下**容器**可以指我们**运行的这组隔离的进程**，在另一些语境下**也可以表示相对于镜像而言的承载某个具体容器的根目录。**

<img src="assets/cc9e8aa94e1bd4d449237b94d5f06142.webp" width="500" />

那我们现在修改一下我们整体的代码结构，让镜像都存放在 images 目录下，每次启动容器时创建的目录都放在 containers 目录下。

修改后的目录大概是这个样子。

```bash
[shanke ~] # tree .
├── containers
│   ├── busybox1
│   └── busybox2
├── images
│   ├── busybox.tar
│   └── hello.tar
├── init_net.sh
```







### 容器和镜像分家

我们每次启动一个新的容器，都需要从镜像中拷贝一份。

但是我的容器进程退出之后，就没有办法保留上一次容器中修改的内容了。

当然，上一次修改的内容实际上是在的，只不过我们没有设计出一个方便的机制，让下一次可以再次「进入」上一次退出的容器里。

那其实很简单，只需要完成三件事即可。

- 第一，每次从 images 镜像文件中拷贝出并存放在 containers 下的目录，都给起一个 ID 作为目录名，作为容器 ID（镜像也可以起一个 ID 但我们这里先不考虑了）
- 第二，设计一个进入容器的命令，实际上就是以 containers 下的 ID 名称的目录作为容器根路径，来启动一个容器进程而已。
- 第三，设计一个查看所有容器信息的命令，实际上相当于 ls 一下 containers 目录。





最终的效果期望如下：

<img src="assets/38ccb9a0ff08745e0420c93fc1891ced.webp" width="500" />

- 通过 `skdocker ps` 命令可以查看所有的容器，其实就是遍历了 containers 目录下的目录名。
- 通过 `skdocker run` 命令可以从 images 目录下把镜像压缩包解压出来到 containers 目录下，并以此为根目录启动一个容器。
- 通过 `skdocker exec` 命令可以进入一个已经创建好的容器中，实际上就是以 containers 目录下的某个目录为根目录创建一个容器进程。





### C代码实现

源代码在 `demos/07-01-copy-image.c` 中，下面列出主要的逻辑。

- 首先对 main 方法进行改造，只处理识别命令字符串的功能。

  ```c
  int main(int argc, char *argv[]) {
      if (strcmp(argv[1], "run") == 0) {
          skdocker_run(&argv[2]);
      } else if (strcmp(argv[1], "exec") == 0) {
          skdocker_exec(&argv[2]);
      } else if (strcmp(argv[1], "ps") == 0) {
          system("tree -L 1 containers");
      }
      else {
          fprintf(stderr, "Usage: skdocker run|exec|ps\n");
          return EXIT_FAILURE;
      }
      return 0;
  }
  ```

- 其中 ps 命令简单地用 tree 脚本实现，查下 containers 目录名即可。

- run 命令单独用 `skdocker_run` 方法实现，即简单地找到 images 下面的 tar 包并解压到 containers 目录下，然后用时间戳生成一个目录名作为容器 ID。

  ```c
  void skdocker_run(char **argv) {
      char image_file[256]; 
      char container_rootfs[256];
      char timestamp_str[20];
  
      snprintf(image_file, sizeof(image_file), "images/%s.tar", argv[0]);
      systemf("tar -xf %s -C containers", image_file);
  
      get_current_timestamp(timestamp_str, sizeof(timestamp_str));
      systemf("mv containers/%s containers/%s", argv[0], timestamp_str);
  
      argv[0] = timestamp_str;
      skdocker_exec(argv);
  }
  ```

- `skdocker_run` 方法最后调用了 `skdocker_exec` 方法，也就是说当容器的目录创建好之后，后面的流程就和进入一个容器一样了。我们看下 `skdocker_exec` 方法的实现。

  ```c
  void skdocker_exec(char **argv) {
      pipe(pipefd);
      // 创建新进程的命名空间标识
      int flags = CLONE_NEWUTS | CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET | CLONE_NEWIPC;
      // 创建新进程，新进程在上述独立的命名空间下
      char container_rootfs[256];
      snprintf(container_rootfs, sizeof(container_rootfs), "containers/%s", argv[0]);
      argv[0] = container_rootfs;
      int pid = clone(child, malloc(4096) + 4096, flags | SIGCHLD, argv);
      printf("父进程创建子进程完毕，子进程 pid = %d\n", pid);
      // 设置 cgroup
      cfcgroup(pid);
      printf("父进程设置 cgroup 完毕\n");
      // 设置网络
      cfnet(pid);
      close(pipefd[1]);
      // 等待子进程结束的信号
      waitpid(pid, NULL, 0);
  }
  ```






### 再次优化写时复制

如果某个容器启动后对目录里的内容没有任何修改，那么就一直复用同一个目录。什么时候有了修改，再把目录单独复制出一份来。

这能解决第一个问题，也就是目录一模一样的时候，不再浪费空间。

但如果目录几乎一模一样，能不能也做到只记录增量的修改，而底层的记录用于只有一份呢？

这就要介绍到 UnionFS 技术。







### UnionFS 联合文件系统

UnionFS（联合文件系统）是一种**基于文件系统的分层存储技术**，它可以将多个目录（称为分支）合并成一个目录呈现给用户，而实际的数据保留在各自的分支中。

它允许多个文件系统或目录以只读或读写的方式合并，并且可以通过分层的方式使得上层可以覆盖下层的内容，而不实际修改下层的数据。

UnionFS 的核心概念有三点：

1. **分层（Layers）**：每一层都是一个只读的文件系统，它们可以在运行时被合并。通过这种分层设计，多个层的内容可以共享，避免重复存储。
2. **写时复制（Copy-on-Write, COW）**：当需要修改某个文件时，系统不会直接修改原始的只读层文件，而是会将需要修改的文件复制到可写的层（通常是最顶层），然后在该层进行修改。
3. **合并视图**：用户看到的文件系统是通过合并多个层形成的虚拟视图，虽然物理上文件分布在不同层，但对用户来说，整个文件系统表现为一个统一的结构。





不废话，直接上手实验。

联合文件系统的实现有很多种，有早期的 aufs 和现在的 `overlayfs`，我们使用 `overlayfs` 来做实验。

<img src="assets/1c63da93bef56725e18b10c5ca53fd30.webp" width="600" />

这里关注 lower upper 和 merge 目录，**使用 overlayfs 来挂载 merge 目录**，里面的内容是由 lower 和 upper 目录加起来的，可以先直观这样理解。

<img src="assets/b9c0040e443ec3896224e032d61e1f74.webp" width="500" />

接下来我们**分别做三个实验。**

- 一、在 merge 目录下新增一个文件，可以看到 upper 目录下也出现了。

  <img src="assets/b3a15c44ae3aa714ae911c8643ba10aa.webp" width="500">

- 二、在 merge 目录下对 upper.txt 进行修改，可以看到 upper 目录下的也被修改了。

  <img src="assets/ec434cdd2396ff3a270ae08c4f830850.webp" width="500"/>

- 三、在 merge 目录下对 lower.txt 进行修改，可以看到 upper 目录下新增了个 lower.txt 文件，并且内容是刚写进去的。然而 lower 目录下完全不变。

  <img src="assets/275946c0cc7eea08097ac291ca1a9bb3.webp" width="500" />

那总结起来也很简单。upper 层的目录一直跟着 merge 层改变，最终合并的目录 upper 层会覆盖 lower 层。





**总结，具体来说:**

- 新增文件时是这样。

  <img src="assets/6dfcf7dfa5750aa809a4e911312d8230.webp" width="500"/>

- 修改来自 upper 的文件时是这样。

  <img src="assets/8b7c9046801cc2583a6920070522191f.webp" width="500"/>

- 修改来自 lower 层的文件时会触发写时复制，复制到 upper 层并覆盖 lower 层文件。

  <img src="assets/0b0bc6c764b00f16125096c312f2b982.webp" width="500"/>







### 用联合文件系统来设计镜像和容器

答案似乎呼之欲出了，只需要把上面的目录改个名就好了。

<img src="assets/08c914b10827ab43981ea31b7b1e9fb0.webp" width="600">

进一步的，你的镜像层也可以进一步按照 upper 和 lower 递归地分层。

<img src="assets/6911c0fe5b5cf0c3d8fc8c0be5095b7b.webp" width="600">

这就利用 UnionFS 实现的镜像与容器的分离！

代码非常简单，就不在文中展开了，可以参考：`demos/07-02-copy-unionfs.c`

我把 Makefile 也编辑好了，直接下载代码执行 `make busybox` 即可体验最终的效果

<img src="assets/f9e791c5d43d3a280c6e55433f88edc1.webp" width="600"/>











## container

**容器的本质就是一组隔离的进程**

我们的做法是，就单纯创建出一个进程，然后给这个进程打各种标签，比如修改它的根路径，打上 namespace 和 cgroup 等标签并写上值，我们操作的一直是一个个进程本身，根本没有所谓的一个叫容器一样的东西存在，让我们把进程放进去。

所以我想到了另两种解释，可以更好解决我的一个困惑：为什么这个东西叫容器？





### 鸭子测试

“If it looks like a duck, swims like a duck, and quacks like a duck, then it probably is a duck”，这句话中文翻译为“如果一个东西看起来像鸭子，走起来像鸭子，叫起来像鸭子，那它就是鸭子”。

它通常被称为“鸭子测试”（Duck Test），是一种非正式的推理方法，通过观察一个事物的表面特征和行为来判断它的本质。

这句话背后的逻辑非常简单：**根据外在表现判断事物的本质**。它的核心思想是，我们可以通过对事物特征的直接观察，来推断该事物的真实身份或本质，即使没有足够的直接证据。这种推理方法强调经验和直觉，属于**归纳推理**的一种。

换回我们的场景。

虽然我们是通过操作一个个进程，最终容器的本质也是一组隔离的进程。但这些东西看起来像容器，听起来像容器，用起来也像容器，那他就是个容器，就没什么不妥了。







### 缸中之脑

缸中之脑（Brain in a Vat）是一个著名的哲学思想实验，通常用于讨论怀疑论（skepticism）、实在论（realism）、意识和感知的问题。这个思想实验提出了这样一个假设：

假设你的大脑被移植到了一个营养液缸中，通过计算机和外部设备连接，向你的大脑传输电信号，这些信号完全模拟你平时通过感官接收到的信号。你能通过你的感觉来判断你是一个“缸中之脑”，还是生活在真实世界中的人吗？

这个问题的核心是，如果你生活在这样一个虚拟世界里，所有的感知和体验都通过外部刺激（计算机提供的信号）产生，那你怎么知道你感知到的世界是真实的，而不是虚拟的？换句话说，你如何知道自己不是一个“缸中之脑”？

换回我们的场景。

身处在容器中的用户，他不论做什么操作，都无法证明自己是在一个容器进程中，还是在一台独立的宿主机上。这正类似于处在虚拟世界的缸中之脑！







课程代码地址：https://gitee.com/wuliaodeshanke/shanke-simple-docker