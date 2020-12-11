---
title: 为什么go/java在写socket时要给fd加锁
date: 2020-12-10 20:43:46
tags:
---

##  为什么go/java在写socket时要给fd加锁？



​	在今年夏天，就有过这样的疑惑，那时候作者在写一个go的rpc,bench结果显示竞争一个fd的lock占了阻塞时间的99.9%

![](/imgs/image-20201210174052784.png)

这时候作者就有疑问了,为什么要加这个锁呢？下面是作者解决这个问题的几个步骤:

####  1.检查write系统调用是否线程安全:(有点南辕北辙了，应该先扒go的代码的)

​	这里作者写了个C的简单程序,比较短，很久之前写的没有留存，就不贴了。大概的内容就是开了四个写socket线程一个读socket线程。

验证能否保证写的原子性。

​	首先开一个读线程监听8080端口，将从server socket读到的信息打印出来，然后主线程连接到8080端口,开四个写线程分别循环的向此client socket写入[1,2,3],发现读线程读到的字节序仍然是123123123....没有破坏读写的原子性。

​	这就很奇怪了，既然写是原子的,那为啥子go要在写的时候加锁呢？（这里作者漏了一个点,在下面会讲）

#### 2.扒go代码

​	从上面的pprof可以看出来,竞争写锁的操作在/internal/poll/fd_unix.go:Write

```go
func (fd *FD) Write(p []byte) (int, error) {
	if err := fd.writeLock(); err != nil {
		return 0, err
	}
	defer fd.writeUnlock()
  ...
  for {
		max := len(p)
		if fd.IsStream && max-nn > maxRW {
			max = nn + maxRW
		}
		n, err := ignoringEINTR(func() (int, error) { return syscall.Write(fd.Sysfd, p[nn:max]) })
		if n > 0 {
			nn += n
		}
		if nn == len(p) {
			return nn, err
		}
		if err == syscall.EAGAIN && fd.pd.pollable() {
			if err = fd.pd.waitWrite(fd.isFile); err == nil {
				continue
			}
		}
		if err != nil {
			return nn, err
		}
		if n == 0 {
			return nn, io.ErrUnexpectedEOF
		}
	}
}
```

##### 重要背景:go连接的tcp socket是非阻塞的

每次向fd写数据时，都会先竞争这个连接的写锁然后再往里面写数据，由于在建立tcp连接，构造socket时会设置为非阻塞:

在/net/sock_posix.go 19行socket函数调用了sysSocket，在sysSocket里设置了nonBlock

下面是Linux平台的sysSocket

```go
func sysSocket(family, sotype, proto int) (int, error) {
   s, err := socketFunc(family, sotype|syscall.SOCK_NONBLOCK|syscall.SOCK_CLOEXEC, proto)
  ...
}
```

其中socketFunc是平台绑定的`socket`系统调用,可以看出来,设置了包括NONBLOCK的socket选项,标记此socket是非阻塞的。

**设置这个选项后，单次write系统调用如果遇到缓冲区满等情况会直接返回EAGAIN err code与已经写入的长度，而不会阻塞等待。**

**并且，在之前用C语言写的多线程写程序中，我们没有设置这个NONBLOCK选项，所以我们每次写都会阻塞直到传入的所有数据写入成功。**

有了上面的背景，举个形象的例子，比如我们仍要4个写线程，仍循环的写1，2，3。如果设置了NONBLOCK这个选项，每个写线程为了保证能写完1，2，3。需要用类似

```  go
for{	
  	n, err := syscall.write(fd,buf[written:])
		if n > 0 {
			written += n
		}
		if written == len(buf) {
			return written, err
		}
		if err == syscall.EAGAIN  {
				continue
		}
	}
```

的循环来写入,假设我们写完1，`write`就因为缓冲区满返回了，我们只能走`err == syscall.EAGAIN  `然后下个循环节再去`write`

如果这时候别的线程写了个1那么最终读到的序列可能是123112323。整个依赖此序列的读逻辑都会因此崩掉。。

所以go需要在写入的时候为fd加锁。



这时候，又会引入一个新问题,可能会在后面的文章中给出答案:

#### 为什么go使用非阻塞的socket?