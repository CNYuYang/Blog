# RPC和线程

## 为什么用 Go

1. **语法先进**。在语言层面支持线程（goroutine）和管道（channel）。对线程间的加锁、同步支持良好。

2. **类型安全（type safe）**。内存访问安全（memory safe），很难写出像 C++ 一样内存越界访问等 bug。

3. **支持垃圾回收（GC）**。不需要用户手动管理内存，这一点在多线程编程中尤为重要，因为在多线程中你很容易引用某块内存，然后忘记了在哪引用过。

4. **简洁直观**。没 C++ 那么多复杂的语言特性，并且在报错上很友好。

## 线程（Threads）

 线程为什么这么重要？因为他是我们控制并发的主要手段，而并发是构成分布式系统的基础。在 Go 中，你可以将 goroutine 认为是线程，以下这两者混用。 每个线程可以有自己的内存栈、寄存器，但是他们可以共享一个地址空间。

## goroutine

```go
package main
 
import (
 
"fmt"
 
"time"
 
)
 
func printString(value string) {
 
for i := 0; i < 10; i++ {
 
fmt.Println(value)
 
time.Sleep(time.Second) //0.1 second
 
}
 
}
 
func main() {
 
go printString("A")
 
go printString("B")
 
time.Sleep(time.Second * 10)//暂时挂起主线程，让goroutine的两个线程跑完
 
}
```

输出结果：ABABBAABABBABABAABAB

两个线程交叉输出，说明是以并发形式执行，这里为什么说是并发而不是并行，因为go语言默认是单核执行线程。如果想要多核并行执行，需要在启动goroutine之前首先调用以下语句配置cpu核数：

runtime.GOMAXPROCS()

自从Go 1.5开始， Go的GOMAXPROCS默认值已经设置为 CPU的核数，我们不用手动设置这个参数。

## go语言的线程通信

无论是哪种编程语言，实现多线程之间的通信方式无非是共享数据和消息两种方式。

### 共享数据

```go

package main
 
import (
 
"fmt"
 
"sync"
 
"time"
 
)
 
var counter int = 0
 
func Count(lock *sync.Mutex) {
 
lock.Lock()
 
counter++
 
fmt.Println(counter)
 
lock.Unlock()
 
}
 
func main() {
 
lock := &sync.Mutex{}
 
for i := 0; i < 5; i++ {
 
go Count(lock)
 
}
 
time.Sleep(time.Second * 5)
 
}
```

输出结果：1 2 3 4 5

此时，多个线程共享数据counter，实际上当业务逻辑比较复杂并且共享数据比较多的情况下，使用这种方式是一件十分头疼的事情。我们这里暂且不提这些。

### 消息机制实现多线程通信

go语言使用channel在两个或多个线程之间传递消息。channel翻译成中文就算通道的意思，我们直接看代码：

```go
package main
 
import (
 
"fmt"
 
"sync"
 
"time"
 
)
 
func WriteIn(ch chan int) {
 
for i := 0; i < 10; i++ {
 
ch <- i
 
}
 
}
 
func ReadOut(ch chan int) {
 
for i := 0; i < 10; i++ {
 
value := <-ch
 
fmt.Println(value)
 
}
 
}
 
func main() {
 
ch := make(chan int)
 
go WriteIn(ch)
 
go ReadOut(ch)
 
time.Sleep(time.Second * 5)
 
}
```

输出：1 2 3 4 5 6 7 8 9，通过chanel在不需要锁的情况下实现了数据的共享，chanel的用法中，常见的操作包括写入和读出，写入和读出都会导致线程的阻塞，也就是说数据写入后线程会等待另一个线程的读取，而读取之前如果没有数据写入也会进入阻塞状态等待数据写入。

## go语言的线程同步

### 使用互斥锁实现线程同步

互斥锁是最简单的一种锁类型，同时也比较暴力，当一个goroutine获得了锁之后，其他goroutine就只能乖乖等到这个goroutine释放该锁。go语言使用sync.Mutex实现互斥锁。

代码如下：

```go
package main
 
import (
 
"fmt"
 
"sync"
 
"time"
 
)
 
type MutexInfo struct {
 
mutex sync.Mutex
 
infos []int
 
}
 
func (m *MutexInfo) addInfo(value int) {
 
m.mutex.Lock()
 
m.infos = append(m.infos, value)
 
m.mutex.Unlock()
 
}
 
func main() {
 
m := MutexInfo{}
 
for i := 0; i < 10; i++ {
 
go m.addInfo(i)
 
}
 
time.Sleep(time.Second * 5)
 
fmt.Println(m.infos)
 
}
```

输出：[0 1 2 3 4 5 6 7 8 9]

我们通过多次运行发现，输出的结果并不总是从0到9按顺序输出，说明创建的10个goroutine并不是有序的抢占线程的执行权，也就是说这种同步并不是有序的同步，我们可以让10个goroutine一个一个的同步执行，但是并不能安排执行次序。

运行到这里，假如我们注释掉同步锁的代码为发生什么？

我们将addInfo方法修改如下：

```go
func (m *MutexInfo) addInfo(value int) {
 
//m.mutex.Lock()
 
m.infos = append(m.infos, value)
 
//m.mutex.Unlock()
 
}
```

运行代码，输出：[1 0 2]

结果是不是出乎意料？为什么写了10个输入，只有3个值输入成功？这时候我们不得不解释线程的另一个概念，那就是线程安全。

我们先看下go语言中slice的append过程，使用append添加一个元素时，可能会有两步来完成：先获取当前切片数组的容量，比如当前容量是2，然后在新的存储区开辟一块新的存储单元，容量为2+1，并将原来的值和新的值存入新的存储单元。在没有同步锁的情况下，如果两个线程同时执行添加元素的操作，这时候可能只有一个被写入成功。这种情况就是非线程安全，相比之下，如果同时对一个int类型数据进行操作，就不会出现这种非线程安全的情况。

### 读写锁实现线程同步

go语言提供了另一种更加友好的线程同步的方式：sync.RWMutex。相对于互斥锁的简单暴力，读写锁更加人性化，是经典的单写多读模式。在读锁占用的情况下，会阻止写，但不阻止读，也就是多个goroutine可同时获取读锁，而写锁会阻止其他线程的读写操作。

代码如下：

```go
package main
 
import (
 
"fmt"
 
"sync"
 
"time"
 
)
 
type MutexInfo struct {
 
mutex sync.RWMutex
 
infos []int
 
}
 
func (m *MutexInfo) addInfo(value int) {
 
m.mutex.Lock()
 
defer m.mutex.Unlock()
 
fmt.Println("write start", value)
 
m.infos = append(m.infos, value)
 
fmt.Println("write start end", value)
 
}
 
func (m *MutexInfo) readInfo(value int) {
 
m.mutex.RLock()
 
defer m.mutex.RUnlock()
 
fmt.Println("read start", value)
 
fmt.Println("read end", value)
 
}
 
func main() {
 
m := MutexInfo{}
 
for i := 0; i < 10; i++ {
 
go m.addInfo(i)
 
go m.readInfo(i)
 
}
 
time.Sleep(time.Second * 3)
 
fmt.Println(m.infos)
 
}
```

输出结果：

> read start 0
read end 0
write start 1
write start end 1
read start 4
read end 4
read start 9
read end 9
read start 3
read start 1
read start 7
read end 7
read start 2
read start 8
read end 8
read end 1
read start 6
read end 6
read start 5
read end 5
read end 3
read end 2
write start 0
write start end 0
write start 2
write start end 2
write start 3
write start end 3
write start 4
write start end 4
write start 5
write start end 5
write start 6
write start end 6
write start 7
write start end 7
write start 8
write start end 8
write start 9
write start end 9
[1 0 2 3 4 5 6 7 8 9]

从结果我们可以看出，开始的时候读线程占用读锁，并且多个线程可以同时开始读操作，但是写操作只能单个进行。

### 使用条件变量实现线程同步

go语言提供了条件变量sync.Cond，sync.Cond方法如下：

Wait，Signal，Broadcast。
Wait添加一个计数，也就是添加一个阻塞的goroutine。
Signal解除一个goroutine的阻塞，计数减一。
Broadcast接触所有wait goroutine的阻塞。

代码如下：

```go

func printIntValue(value int, cond *sync.Cond) {
	cond.L.Lock()
	if value < 5 {
		//value小于5时，进入等待状态
		cond.Wait()
	}
	//大于5的正常输出
	fmt.Println(value)
	cond.L.Unlock()
}
 
func main() {
 
	//条件等待
	mutex := sync.Mutex{}
	//使用锁创建一个条件等待
	cond := sync.NewCond(&mutex)
 
	for i := 0; i < 10; i++ {
		go printIntValue(i, cond)
	}
 
	time.Sleep(time.Second * 1)
	cond.Signal()//解除一个阻塞
	time.Sleep(time.Second * 1)
	cond.Broadcast()//解除全部阻塞
	time.Sleep(time.Second * 1)

}
```

运行后先输出满足条件的值：5 6 7 8 9

解除一个阻塞，输出0，解除全部阻塞，输出1 2 3 4

go语言多线程支持全局唯一性操作，即一个只允许goruntine调用一次，重复调用无效。

go语言还支持sync.`WaitGroup`实现多个线程同步的操作。


# 相关笔记

6.824 2020 视频笔记二：RPC和线程 [https://zhuanlan.zhihu.com/p/111450031](https://zhuanlan.zhihu.com/p/111450031)

