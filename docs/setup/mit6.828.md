## 实验环境配置

``VirtualBox``虚拟机为载体，安装Ubuntu

```sh
$ uname -a
Linux eliot-VirtualBox 5.11.0-36-generic #40~20.04.1-Ubuntu SMP Sat Sep 18 02:14:19 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
```

下载实验框架

```sh
$ git clone https://pdos.csail.mit.edu/6.828/2018/jos.git mit6.828
```

安装``toolchain``

先检查本机安装

```sh
$ gcc -m32 -print-libgcc-file-name
/usr/lib/gcc/x86_64-linux-gnu/9/libgcc.a
```

不然则安装相关工具

```sh
sudo apt-get install -y build-essential gdb
```

安装32位支持

```sh
sudo apt-get install gcc-multilib
```

或者直接一站式解决

```sh
sudo apt-get install -y build-essential libtool libglib2.0-dev libpixman-1-dev zlib1g-dev git libfdt-dev gcc-multilib gdb
```

对于``qemu``虚拟机，安装课程推荐的定制版本为佳：

```sh
git clone git@github.com:mit-pdos/6.828-qemu.git
```

开始进行配置

先安装配置需要的``python2.7``

```sh
sudo apt-get install python2.7
```

```sh
./configure --disable-kvm --target-list="i386-softmmu x86_64-softmmu" --python=python2.7
```

开始编译安装

先进入root用户

```sh
su root
```

开始安装

```sh
make && make install
```

可能遇到的错误：

- 错误1

![image-20210930100241590](C:\Users\ELIO\Desktop\MIT6.828\docs\setup\fig\image-20210930100241590.png)

解决方法

```sh
.../6.828-qemu$ vim Makefile
```

更改``Makefile``，在最后一行添加

```sh
QEMU_CFLAGS+=-w
```

- 错误2

![image-20210930100509215](C:\Users\ELIO\Desktop\MIT6.828\docs\setup\fig\image-20210930100509215.png)

解决方法

```sh
.../6.828-qemu$ cd qga/
.../6.828-qemu/qga$ vim commands-posix.c 
```

在头文件中添加

```sh
#include <sys/sysmacros.h>
```

- 错误3

![image-20210930100741216](C:\Users\ELIO\Desktop\MIT6.828\docs\setup\fig\image-20210930100741216.png)

解决方法

```sh
.../6.828-qemu$ vim config-host.mak
```

删除其中的``-Werror``

- 错误4

![image-20210930100859424](C:\Users\ELIO\Desktop\MIT6.828\docs\setup\fig\image-20210930100859424.png)

解决方法

进入root用户模式

```sh
su root
make && make install
```

还有其他错误的，可以自行Google解决.

进入实验的文件夹下：

```sh
make
```

不出意外会出现如下显示

![image-20210930101230981](C:\Users\ELIO\Desktop\MIT6.828\docs\setup\fig\image-20210930101230981.png)

之后运行虚拟机

```sh
make qemu
```

![image-20210930101323315](C:\Users\ELIO\Desktop\MIT6.828\docs\setup\fig\image-20210930101323315.png)

这样``MIT6.828``的实验环境便配置成功了。

### 参考资料

https://pdos.csail.mit.edu/6.828/2018/labguide.html

https://pdos.csail.mit.edu/6.828/2018/labs/lab1/

https://www.cnblogs.com/gatsby123/p/9746193.html

https://github.com/woai3c/MIT6.828/blob/master/docs/install.md