---
title: channel & selelct
date: 2020-11-23
tags: 
- pic
---



# 并发编程模型



![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/cf91b4c995dd4ee2bbcf8662dd7d3e9120211227192856.jpeg)

并发编程的意义，这个不需要多说了，大流量，高并发全靠它，性能上来了，稳定性也就不言而喻了

并发的两个基础部分：

1. 并发调度单位，concurrency unit 
2. 并发模型，concurrency model

## Concurrency unit

并发调度单位讲究 轻！快！

unit占用轻！unit切换快！

### 进程作为Unit

进程拥有独占的内存和指令流，是一个应用的包装，但是进程作为并发基本单元有如下问题：

1. 资源占用过大
    1. 每个进程占用的内存太大了，进程携带了自己的虚拟内存页表，文件描述符等
2. 不能发挥多核的性能
    1. 进程不能很好的发挥多核机器的性能，常常出现一个核跑，多个核看的现象
    2. <img src="https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/image-20211117181946354.png" alt="image-20211117181946354" style="zoom: 50%;" />
3. 进程切换消耗过大
    1. 进程切换需要进行系统调用，涉及到内存从用户态拷贝至内核态
    2. 保存当前进程的现场，并且恢复下一个进程



<img src="https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/image-20211117192847504.png" alt="image-20211117192847504" style="zoom:50%;" />

### 线程作为Unit

线程较轻量级，一个进程可以包含多个线程，则每个最小并发粒度的资源占用要小很多，且同一个进程内线程间切换只需要对指令流进行切换即可。

<img src="https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/image-20211118005842113.png" alt="image-20211118005842113" style="zoom:50%;" />

但是，进程间切换仍需要进入内核进行，仍然存在大量的并发切换消耗

### 协程作为Unit

协程，也叫做**用户态线程**，它规避了最后一个问题，切换消耗过大的问题，无需通过系统调用进入内核进行切换，协程所有的生命周期均发生在用户态。

![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/bVcTwI820211227195035.png)

因为协程的优点，协程类编程也开始越来越火了。比较有代表性的有Go的goroutine、Erlang的Erlang进程、Scala的actor、windows下的fibre（纤程）等，一些动态语言像Python、Ruby、Lua也慢慢支持协程。

**但是** 语言引入协程作为并发调度单位，需要实现自己的协程调度器、并提供协程间通信的方式等一系列支持模块，相较于传统的基于进程线程的并发方式，需要实现很多额外的功能组件。**实现较复杂。**



## Concurrency model

总体来看，目前能找到的最轻量的调度单元就是协程了，虽然实现起来有些麻烦，但是现代语言也越来越多的引入协程了。

那么解决了并发单元的问题后，我们再研究下并发模型，为什么需要并发模型呢，因为**并发就意味着竞争**：对内存的竞争，对算力的竞争等，那么如何降低竞争带来的性能损耗，就需要并发模型的设计了。简单来说，并发模型就是指导并发单元以何种方式处理竞争，尽量减少竞争带来的性能损耗。简单来说，就是**定义了并发单元间的通信方式**。

### 共享内存+锁

<img src="https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/image-2021122720381900520211227203819.png" alt="image-20211227203819005" style="zoom:50%;" />

最经典的模型，通过锁来保护资源，多个并发单元访问资源前首先争夺锁，然后再去访问资源。没抢到锁的unit则阻塞等待。

这个应该是目前最常用的了，也最符合直觉，但是可以明显看到，在竞争时会产生**阻塞耗时**。

**这就是常说的使用共享内存来进行通信**

### 函数式编程

**既然基于共享内存通信会产生大量的竞争，那么函数式编程的通信思想是，在并发单元执行过程中不进行通信，只在最后大家都执行完后统一对结果做收集和汇总**



<img src="https://coolshell.cn/wp-content/uploads/2013/12/yoda-lambda-204x300.png" alt="img" style="zoom: 50%;" />

函数式编程的特性：

1. 不可变数据，默认是变量是不可变的，如果你要改变变量，你需要把变量copy出去
2. 函数对于Input A一定会返回Output B，即**函数内部没有状态**，不会对全局变量进行修改，运行时涉及的变量都是局部变量
3. 这么一来，每个函数对输入负责，只会访问局部变量，全局不存在资源竞争

基于函数式编程模型作为并发模型的话，性能会很高，但是会产生额外的大量的局部变量

**代表语言：clojure**

#### 举个例子：

s3e在设计之初，提供了一套SDK，目的是帮助业务建模和模型可视化，大体是这样的，将一个业务功能节点抽象为了workflow，workflow中的每个task state对应一个函数，为了降低使用成本，各个函数的签名都是一致的

```go
func Action(ctx context.Context, db *Databus) (*Databus, error)
```

每个函数都会对Databus做一些自己的修改，这个修改是全局Action可见的（因为Databus传的是指针类型），因此如果存在并发的节点，会存在对全局变量的锁竞争。

但是在接入业务需求时，这套设计还好，不会产生太大的问题，但是在接入拦截系统时，由于拦截系统的比较大的诉求是希望引入并发带来对耗时的优化，如果仍然采用这种粗粒度锁的方式，竞争会比较大，可预见的性能优化不会太明显，如图

![image-20211228182759402](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/image-2021122818275940220211228182800.png)

```go
type Data struct {
	input 		*Input
  collector *ChekerCollector
}

type CheckerCollector struct {
  lock 			sync.Lock
	data map[string]*CheckRes
}

func (cc *CheckerCollector) Report(key string, res *CheckRes) {
  cc.lock.Lock()
  map[key] = res
  cc.lock.Unlock()
}
```



那么，采用函数式编程的思路，我们把databus尽量减少写操作，将需要写的字段分配到每个并发节点的运行时局部变量中，然后再对每个并发节点的结果做统一的收集，可以很好的减少并发竞争

![image-20211228182726909](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/image-2021122818272690920211228182727.png)



### Actor

那么，我们回归到通信本身，有没有更好的通信方式呢？

Actor的主要思路是，每个并发单元抽象为actor，每个actor都拥有一个邮箱**，所有actor间的通信都会异步的发送到对方的邮箱中**，这样**解耦了actor**之间的关系，各个actor都能按照自己的步调进行执行，但是他们会按照邮箱中消息的发送顺序来依次处理消息，当且仅当前一个消息处理完成后，才会开始下一个消息处理，即**保障了消息的时序性**。

<img src="https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/image-2021122819182969720211228191830.png" alt="image-20211228191829697" style="zoom:67%;" />

这样的话，在并发单元执行过程中，也不会存在锁资源的竞争，但是由于发送过程是异步的，即只将消息放入目标actor的邮箱中即完成了发送操作，但是消息什么时候会被目标actor处理，则是不可预测的。

![image-20211228205442883](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/image-2021122820544288320211228205443.png)



**代表语言：erlang，scala的akka库**



### CSP

csp的降低竞争的思想大体和actor保持一致，但是在消息发送上则采用了同步的方式，且与actor不同的是，csp关注的不是发送接受者，而是发送的媒介。

![image-20211228205501998](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/image-2021122820550199820211228205502.png)

并发单元间通过一个FIFO队列（channel）来进行通信，而不是直接和目标单元进行通信。



#### actor和csp的区别

![image-20211228205628395](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/image-2021122820562839520211228205629.png)

1. actor的并发单元之间直接通信，csp则通过channel通信，后者显然耦合更松散
2. csp的消息交换是同步的，actor则是完全异步，可以在任意时间点发送消息到任何并发单元（甚至不存在的），并且actor可以自由选择处理哪些消息
3. 相应的，csp的channel空间可以使有限的，而actor的邮箱理论上是需要无限大的
4. actor关注的是并发单元，而csp关注的则是channel



### 现实

现实上，几乎所有并发编程语言**都支持锁+共享内存**的形式进行并发单元通信，而对于支持函数式编程，actor和csp等概念的大部分语言，并没有严格完全按照模型定义实现，都在使用上或多或少的做了一些折中。



# channel

步入正题！

那么，显然golang深受csp模型的影响，channel和gorutinue是golang中的一等公民。

但是它又没有完全按照CSP理论中的channel来实现，CSP中的channel是一个纯粹的同步信道，而go channel不仅支持同步式通信，而且支持非同步式信道。

我们已经知道，channel本质上就是一个**有锁的并发安全的FIFO消息队列**，他负责在gorutinue之间传递消息。



## Doc



### 类型

```go
ChannelType = ( "chan" | "chan" "<-" | "<-" "chan" ) ElementType .
```

包含三种类型的定义，可选的'<-'代表了channel的方向，如果没有指定方向，channel就是双向的，可以接受数据，也可以发送数据。

举个例子：

```go
		var ch1 chan int   // 声明一个传递整型的通道
    var ch2 chan bool  // 声明一个传递布尔型的通道
    var ch3 chan []int // 声明一个传递int切片的通道
```

### 创建

channel是引用类型，channel的空值为nil

```go
var ch chan int
fmt.Println(ch) // <nil>
```

所以，通道在初始化后，还需要使用make函数进行初始化后才能使用

```
make(chan 元素类型, [缓冲大小])
```

缓冲区大小可选，举个例子

```go
ch4 := make(chan int)
ch5 := make(chan bool 10)
ch6 := make(chan []int 1)
```



### 发送



```go
ch <- 10 //将10发送到channel中
```



### 接收

```go
x, ok := <- ch //判断ch是否已经关闭
x := <- ch //从chan中接受值并赋值给x
<- ch //从ch中接收值并忽略
```



### 关闭

```go
close(ch)	
```

Tips：

只有在需要通知接收方所有数据已经发送完毕时，才需要显式的调用close函数关闭chan，除此之外，若不对channel进行关闭操作，它是可以被垃圾回收机制回收的，**关闭通道不是必须的**

通道关闭后：

- 对一个关闭的channel再发送值回panic
- 对一个关闭的channel进行接收会一直获取值直到channel为空
- 对一个关闭的channel并且没有值的channel执行接收操作会得到类型的零值
- 重复关闭channel会panic



### 优雅的读取channel

```go
// channel 练习
func main() {
    ch1 := make(chan int)
    ch2 := make(chan int)
    // 开启goroutine将0~100的数发送到ch1中
    go func() {
        for i := 0; i < 100; i++ {
            ch1 <- i
        }
        close(ch1)
    }()
    // 开启goroutine从ch1中接收值，并将该值的平方发送到ch2中
    go func() {
        for {
            i, ok := <-ch1 // 通道关闭后再取值ok=false
            if !ok {
                break
            }
            ch2 <- i * i
        }
        close(ch2)
    }()
    // 在主goroutine中从ch2中接收值打印
    for i := range ch2 { // 通道关闭后会退出for range循环
        fmt.Println(i)
    }
}
```

可以看到👆的例子，存在两种接收channel的方式，并且都处理了channel关闭的情况，按照个人喜好选择吧。



### 查看channel容量

```go
len(ch) //查看ch缓冲区内的元素
cap(ch) //查看ch缓冲区的最大容量
```





## 无缓冲channel





<img src="https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/image-2022010321131727020220103211317.png" alt="image-20220103211317270" style="zoom:67%;" />

无缓冲区的channel又叫做阻塞channel，举个例子

```go
func main() {
	ch := make(chan int)
	ch <- 9527
	fmt.Println("OK")
}
```



上面这段代码👆，执行时会报错吗

```
$ go run main.go
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
        /home/xiaoju/gulf/test/main.go:7 +0x54
exit status 2
```

简单来说，阻塞式的channel是需要先有接收者才能发送的，否则会一直阻塞，改进：

```go
func recv(c chan int) {
    ret := <-c
    fmt.Println("recv ok", ret)
}
func main() {
    ch := make(chan int)
    go recv(ch) // 启用goroutine从通道接收值
    ch <- 10
    fmt.Println("ok")
}
```



反之，当只有发送者时，发送者也会阻塞，等待接收者到来才能发送，因此，使用无缓冲channel进行通信将导致**发送和接收的goroutine同步化**，因此无缓冲channel也称为**同步channel**。



## 有缓冲的channel

<img src="https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/image-2022010317133827620220103171339.png" alt="image-20220103171338276" style="zoom:67%;" />

我们可以在channel初始化时声明其容量

```go
func main() {
    ch := make(chan int, 1) // 创建一个容量为1的有缓冲区通道
    ch <- 10
    fmt.Println("OK")
}
```

只要缓冲区未满，则发送者发送消息不回阻塞，直到缓冲区满了之后，发送者发送消息才会阻塞，反之，缓冲区不空，接收者接收消息不阻塞，缓冲区空，接收者接收消息阻塞。



## 源码总览



### 主结构



![image-20211229113021690](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/image-2021122911302169020211229113022.png)



```go
type hchan struct {
	qcount   uint           // 队列中的所有元素个数
	dataqsiz uint           // ring buffer的大小
	buf      unsafe.Pointer // ring buffer 数组实现
	elemsize uint16					// 元素大小
	closed   uint32					// 是否关闭
	elemtype *_type // 元素类型
	sendx    uint   // 发送的索引
	recvx    uint   // 接受索引
  recvq    waitq  // recv 等待队列 (<-chane)
  sendq    waitq  // send 等待列表 (chan<-)

	lock mutex //锁
}
```

为什么使用ringbuffer来存元素？首先，我们需要实现一个FIFO的队列，

那么第一反应是链表，新增和删除节点都是O(1)，但是链表在send操作时，需要重新分配内存创建一个链表节点。recv操作时，recv后的链表节点需要GC去识别与回收内存。这明显有点奢侈。

<img src="https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/image-2022010420102572120220104201026.png" alt="image-20220104201025721" style="zoom:50%;" />



那么如果使用ringbuffer的话，首先，一次分配内存后，无论send还是recv操作，都不存在内存分配操作。而且由于数组的大小是固定的，可以直接将ringbuffer和hchan放入连续内存中，提高访问速度。

<img src="https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/image-2022010421265878120220104212659.png" alt="image-20220104212658781" style="zoom:50%;" />











```go
type waitq struct {
	first *sudog
	last  *sudog
}
```









```go
type sudog struct {
    // The following fields are protected by the hchan.lock of the
    // channel this sudog is blocking on. shrinkstack depends on
    // this for sudogs involved in channel ops.
  	//下面的字段，由hchan.lock保护
    g *g

    // isSelect indicates g is participating in a select, so
    // g.selectDone must be CAS'd to win the wake-up race.
  	// isSelect 标识 g 是否正在参与一个select，因此g.selectDone必须以 CAS 的方式来避免唤醒时的race
    isSelect bool
    next     *sudog
    prev     *sudog
    elem     unsafe.Pointer // data element (may point to stack) 指向要发送或者接受的元素数据，可能指向栈

    // The following fields are never accessed concurrently.
    // For channels, waitlink is only accessed by g.
    // For semaphores, all fields (including the ones above)
    // are only accessed when holding a semaRoot lock.
  	//以下字段永远不会被并发访问，

    acquiretime int64
    releasetime int64
    ticket      uint32
    parent      *sudog // semaRoot binary tree
    waitlink    *sudog // g.waiting list or semaRoot
    waittail    *sudog // semaRoot
    c           *hchan // channel 反向索引
}
```





## 创建

## 发送

## 接受

## 关闭





# select

## 分类

zero-case

uni-case

muilt-case

多个case，上一个case执行过程中，下一个case的chan有数据来了的话，会怎么做
