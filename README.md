# Linux内核5.0系统调用处理过程
学号241，姓名刘乐成，原创作品转载请注明出处 + [https://github.com/mengning/linuxkernel/](https://github.com/mengning/linuxkernel/)。

## 实验环境

VMware Workstation 14 Player虚拟机
Ubuntu 64位系统

## 实验目的
跟踪分析Linux内核5.0系统调用处理过程，并选择系统调用号后两位与学号后两位相同的系统调用进行跟踪分析。

## 实验步骤
1、下载linux5.0内核源码

```
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.0.tar.xz
```

2、解压并编译内核5.0

```
xz -d linux-5.0.tar.xz
tar -xvf linux-5.0.tar
cd linux-5.0
make i386_defconfig
make -j8
```
![编译](https://github.com/llc1024/Linux-5.0-/blob/master/TIM%E6%88%AA%E5%9B%BE20190318222028.png)

3、制作根文件系统，下载并编译menuOS

```
cd ..
mkdir rootfs
git clone https://github.com/mengning/menu.git
cd menu
sudo apt install gcc-multilib
gcc -pthread -o init linktable.c menu.c test.c -m32 -static
cd ../rootfs
cp ../menu/init ./
find . | cpio -o -Hnewc | gzip -9 > ../rootfs.img
```

4、启动menuOS

```
qemu-system-i386 -kernel linux-5.0/arch/x86/boot/bzImage -initrd rootfs.img
```
![menuos](https://github.com/llc1024/Linux-5.0-/blob/master/TIM%E6%88%AA%E5%9B%BE20190318224150.png)
然后打开一个新的终端

```
cd linux-5.0
gdb vmlinux
（gdb）target remote:1234
break start_kernel
c
```

至此，linux5.0内核编译成功。

## 选择系统调用号后两位与学号后两位相同的系统调用进行跟踪分析

我的学号后两位为41，第41号系统调用为dup，函数原型：
```
#include
int dup(int oldfd);
```
dup用来复制oldfd所指的文件描述符。但复制成功时返回最小的尚未被使用的文件描述符。若有错误则返回－1，错误代码存入errno中。返回的新文件描述符和参数oldfd指向同一个文件，共享所有的锁定，读写指针，和各项权限或标志位。

在test.c中增加如下代码
```
int duptest(int argc, char *argv[]){
    int fd;
    int new_fd;

    fd = open("./hello.c", O_RDWR | O_CREAT | O_TRUNC, 0666);
    printf("fd = %d\n", fd);

    new_fd = dup(fd);    //复制文件描述符,把fd复制给new_fd
    printf("new_fd = %d\n", new_fd);
    write(new_fd, "hello", 5);

    close(fd);
    close(new_fd);

    return 0;
}

int main()
{
    PrintMenuOS();
    SetPrompt("MenuOS>>");
    MenuConfig("version","MenuOS V1.0(Based on Linux 3.18.6)",NULL);
    MenuConfig("quit","Quit from MenuOS",Quit);
    MenuConfig("time","Show System Time",Time);
    MenuConfig("time-asm","Show System Time(asm)",TimeAsm);
    MenuConfig("duptest","Show duptest",duptest);
    ExecuteMenu();
}
```
重新编译后
![重新编译](https://github.com/llc1024/Linux-5.0-/blob/master/TIM%E6%88%AA%E5%9B%BE20190319130525.png)
调用成功，之后再进行调试。

创建41.c文件
```
#include <stdio.h>

#include <unistd.h>

#include <sys/types.h>
#include<sys/stat.h>
#include<fcntl.h>

int main(){
    int fd;
    int new_fd;

    fd = open("./hello.c", O_RDWR | O_CREAT | O_TRUNC, 0666);
    printf("fd = %d\n", fd);

    new_fd = dup(fd);    //复制文件描述符,把fd复制给new_fd
    printf("new_fd = %d\n", new_fd);
    write(new_fd, "hello", 5);

    close(fd);
    close(new_fd);

    return 0;
}
```

之后编译并进行调试
```
gcc -g 41.c -o 41 -m32
gdb -q
(gdb) file 41
```
![调试1](https://github.com/llc1024/Linux-5.0-/blob/master/TIM%E6%88%AA%E5%9B%BE20190319122426.png)
![调试2](https://github.com/llc1024/Linux-5.0-/blob/master/TIM%E6%88%AA%E5%9B%BE20190319123821.png)

## 系统调用工作机制

系统调用是Linux内核提供的基础服务入口，通过使用这一机制，应用程序可以使用内核的一些专门功能。在分析系统调用之前，以下三点需要了解：
1.系统调用将CPU从用户态切换到核心态，以便访问受保护的内核内存。
2.系统调用的组成是固定的，每个系统调用在内核中都由一个唯一的数字来标识。
3.系统调用的参数传递方式与普通C函数的方式有所不同。
