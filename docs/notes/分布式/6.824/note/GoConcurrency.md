# Go, Threads, and Raft

**注**： goroutine 是一种轻量级的线程，下面 goroutine 和线程两个名词混用，都表示 go 中的线程，虽然他们并不完全等价。

## 内存模型

用线程有两个作用：

1. 提高性能，利用多核。
2. 更优雅的构造代码。

在本课程实验中，我们并不要求使用线程以极尽性能，只要求程序的正确性。

对锁的使用也一样，不要求细粒度的加锁以提升性能，可以大范围加锁以简化代码。

[内存模型](https://link.zhihu.com/?target=https%3A//golang.org/ref/mem)中主要提到了多线程执行时，代码运行的顺序性。主要结论是，单个线程内的可以保证执行效果和代码语句顺序一致，但是跨线程间，如果没有做显式同步（通过锁或者 channel），那么语句执行的先后是没有办法保证的。内存模型大致就讲这么多，本节课的重点在于探讨实验中可能会用到的一些典型*代码模式*。

## 线程和闭包（goroutine && closure）

使用 for 循环 + goroutine 可以很自然的表达 Leader 并行地给 Follower 发送消息的过程。但需要注意循环变量（下例中的 `i`）在子 goroutine 中被引用时，最好事先拷贝一份（一般是通过*函数传参*或者*在循环体内复制*来拷贝）。此外，经常利用 WaitGroup 来在父 goroutine 中阻塞地等待一组子 goroutine 的完结。样例如下：

```go
func main() {
    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(x int) {
            sendRPC(x)
            wg.Done()
        }(i)
    }
    wg.Wait()
}

func sendRPC(i int) {
    println(i)
}
```

## 时间库（time）

使用 go 标准库 time ，可以很方便的实现周期性的做某些事情，比如 raft 中的心跳逻辑。

```go
func periodic() {
    for {
        println("heartbeat")
        time.Sleep(1 * time.Second)
    }
}
```

在 raft 被杀死时，你可能想要结束所有后台线程，可以使用一个共享变量作为循环跳出条件来达到目的。

```go
var done bool
var mu sync.Mutex

func main() {
    println("started")
    go periodic()
    time.Sleep(5 * time.Second) // wait for a while so we can observe what ticker does
    mu.Lock()
    done = true
    mu.Unlock()
    println("cancelled")
    time.Sleep(3 * time.Second) // observe no output
}

func periodic() {
    for done{
        println("tick")
        time.Sleep(1 * time.Second)
        mu.Lock()
        if done {
            return
        }
        mu.Unlock()
    }
}
```

需要注意的是，共享变量 `done` 修改和读取时，都需要使用锁 `sync.Mutex` 包裹起来，这是通过锁 `sync.Mutex` 强制多线程间同步（一个 goroutine `mu.Unlock` 会唤醒另外的 goroutine 正在阻塞 `mu.Lock`）来保证主线程对 `done` 的修改一定能够被子线程看到。否则，如果不做任何同步措施，由于多线程内部的变量缓存问题，go 的内存模型并不严格对共享变量的可见性做保证。

## 锁（mutex）

主要有以下两种情况需要使用锁：

1. 保证多线程间共享变量的可见性。
2. 保证一个代码块的原子性（不会与其他 goroutine 中的语句交替执行）。

当然这两种情况往往是一种情况。

由于 go 内存模型的不可捉摸性（fancy），最好通过加锁来保护所有对共享变量的访问，否则写出的多线程代码很可能会出现一些难以发现和调试的问题。下面是一个使用多线程对未加锁的共享变量进行自增的例子：

```go
func main() {
    counter := 0
    for i := 0; i < 1000; i++ {
        go func() {
            counter = counter + 1
        }()
    }

    time.Sleep(1 * time.Second)
    println(counter) // 大概率不输出 1000
}
```

最后 `counter` 值不为 1000 的原因有两个：

1. `counter = counter + 1` 编译成 CPU 指令后不是原子的，并行运行时可能会产生交错执行。
2. `counter` 修改后可能不能及时被其他线程所看到，从而对旧值加一。

修改后如下：

```go
func main() {
    counter := 0
    var mu sync.Mutex
    for i := 0; i < 1000; i++ {
        go func() {
            mu.Lock()
            defer mu.Unlock()
            counter = counter + 1
        }()
    }

    time.Sleep(1 * time.Second) // 这里用 WaitGroup 更为稳妥，等待1s只能保证大概率正确。
    mu.Lock()
    println(counter)
    mu.Unlock()
}
```

下面看一个 Alice 和 Bob 相互借钱，并试图维持总钱数不变的例子。我们各用一个线程来分别表示 Alice 借给 Bob 钱和 Bob 借给 Alice 钱的过程。

```go
func main() {
    alice, bob := 10000, 10000
    var mu sync.Mutex

    total := alice + bob

    go func() {
        for i := 0; i < 1000; i++ {
            mu.Lock()
            alice -= 1
      mu.Unlock()
      mu.Lock()
            bob += 1
            mu.Unlock()
        }
    }()
    go func() {
        for i := 0; i < 1000; i++ {
            mu.Lock()
            bob -= 1
      mu.Unlock()
      mu.Lock()
            alice += 1
            mu.Unlock()
        }
    }()

    start := time.Now()
    for time.Since(start) < 1*time.Second {
        mu.Lock()
        if alice+bob != total {
            fmt.Printf("observed violation, alice = %v, bob = %v, sum = %v\n", alice, bob, alice+bob) // # violation
        }
        mu.Unlock()
    }
}
```

上述代码会打印出 violation 么？答案是会的。因为观察线程可能在某人借出钱，但是另外一个人没有收到钱的时候打印出 violation。可以从以下两个角度来理解这个问题：

1. **原子性**。出借和借钱应该是一个原子性操作，因此需要使用锁整个包裹起来。否则在中间某个时刻观察，就会产生不一致：钱被借出了，但是还没有收到，即在“控制”。
2. **不变性**（invariant）。锁可以守护不变性，即当获取锁进入临界区后，可能会破坏不变性（借钱，此时钱在“空中”，此时观察会凭空少了一块钱），但是释放锁前恢复（收钱，从“空中”放到另一个人的账户里，两人账户和维持不变）即可。

因此需要将交易过程改为如下：

```go
go func() {
        for i := 0; i < 1000; i++ {
            mu.Lock()
      alice -= 1
            bob += 1
            mu.Unlock()
        }
}()
// 另一个线程也对应修改
```

## 条件（Condition）

在 raft 里，会有一个场景，*Candidate* 向所有 *Followers* 要票，然后根据收集到的票数决定是否成为 *Leader*。

```go
func main() {
    rand.Seed(time.Now().UnixNano())

    count := 0
    finished := 0
    var mu sync.Mutex

    for i := 0; i < 10; i++ {
        go func() {
            vote := requestVote()
            mu.Lock()
            defer mu.Unlock()
            if vote {
                count++
            }
            finished++
        }()
    }

    for { // busy wait
        mu.Lock()

        if count >= 5 || finished == 10 { 
            break
        }
        mu.Unlock()
    }

  mu.Lock()
    if count >= 5 {
        println("received 5+ votes!")
    } else {
        println("lost")
    }
    mu.Unlock()
}

func requestVote() bool {
    time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
    return rand.Int() % 2 == 0
}
```

这种写法有个忙等待，再加上循环过程中不断加锁、释放锁会造成极大的 CPU 占用。

一种很简单的性能提升方法，在忙等待循环中加一个 `time.Sleep(50 * time.Millisecond)`，就能极大的减少 CPU 的占用。

另一种更高效的方式，是使用 `condition`，`condition` 与一个锁绑定，在获取锁之后，通过 `Wait` 来暂时挂起线程，并且释放锁；于是另一个线程就可以通过 `Lock` 获取锁，并且在出临界区前，通过调用 `Broadcast` 来唤醒所有 `Wait` 在此锁上挂起的线程。类似使用一种信号机制，来在多个线程间进行通知锁的释放和获取。

使用 `Condition` 修改上述代码如下：

```go
func main() {
    rand.Seed(time.Now().UnixNano())

    count := 0
    finished := 0
    var mu sync.Mutex
    cond := sync.NewCond(&mu)

    for i := 0; i < 10; i++ {
        go func() {
            vote := requestVote()
            mu.Lock()
            defer mu.Unlock()
            if vote {
                count++
            }
            finished++
            cond.Broadcast()
        }()
    }

    mu.Lock()
    for count < 5 && finished != 10 {
    cond.Wait() // 1. release the mu 2. wait Broadcast() 3. try to acquire the mu again
    }
    if count >= 5 {
        println("received 5+ votes!")
    } else {
        println("lost")
    }
    mu.Unlock()
}
```

需要注意的是，`Condition` 的使用模式要求 `cond.Broadcast` 和 `cond.Wait` 都必须在对应的锁所守护的临界区间内，并且调用 `cond.Broadcast` 后要及时释放锁，否则会引起其他线程的对一个未释放的锁的争抢。

```go
mu.Lock()
// do something that might affect the condition
cond.Broadcast()
mu.Unlock()

//----

mu.Lock()
while condition == false {
    cond.Wait()
}
// now condition is true, and we have the lock
mu.Unlock()
```

此外，`cond.Signal` 每次仅唤起一个调用 `cond.Wait` 进入等待的线程，而 `cond.Broadcast` 会唤起所有等待在相应锁上的线程。当然，前者更高效。

## 通道（channel）

不带缓冲（unbuffered channel）的 channel 通常被用作多线程间进行同步的一种控制手段。

```go
func main() {
    c := make(chan bool)
    go func() {
        time.Sleep(1 * time.Second)
        <-c
    }()
    start := time.Now()
    c <- true // blocks until other goroutine receives
    fmt.Printf("send took %v\n", time.Since(start))
}
```

channel 用在同一个线程内部没有意义，一般都是用作多线程间的通信手段（进行同步或者传递消息）。

而带缓冲的 channel 类似一个同步队列，不过在此次 raft 实验中基本用不到，建议在实验中通过共享变量+锁来进行多线程间消息的同步和互斥。

使用 channel 进行同步，可以发挥类似 `WaitGroup` 的作用：

```go
func main() {
    done := make(chan bool)
    for i := 0; i < 5; i++ {
        go func(x int) {
            sendRPC(x)
            done <- true
        }(i)
    }
    for i := 0; i < 5; i++ {
        <-done
    }
}

func sendRPC(i int) {
    println(i)
}
```

## Raft 死锁

死锁的一个很重要的条件，通俗来说是，占有并等待。即都是*占着自己碗里的不放，还想要对方碗里*，自然就会形成死锁。

一个简单原则是在进行耗时任务时（比如 rpc，IO），要及时释放锁。

## RPC 回应过期

在 Candidate 收回选票结果时，需要判断自己是否仍然在原来的 term 内，以及是否仍然是 Candidate。否则可能选出两个 leader。

```go
if rf.state != Candidate || rf.currentTerm != term {
  return
}
```

不过，其实只检查 currentTerm 就足够了，但是只检查 state 是不够的。

## 调试

DPrintf ：server 编号是一个很重要的字段，建议输出在开头；然后在关注的地方前后加输出语句。

死锁：可以通过 'control+\' 发送一个 SIGQUIT 信号给正在运行中的 go 程序，然后程序就会退出，并且打印出当前各个线程的运行栈。

竞态条件：`go test -race -run 2A` ，使用 `-race` 当 go 检测到竞态发生时打印运行栈。一般多是共享变量被多个线程访问，并且至少有一个访问的地方没有加锁。

## 名词解释

临界区（strict area）：多个线程需要互斥访问的代码段。

忙等待（busy wait）：一直循环，直到条件满足。
