---
title: "【golang】sync.WaitGroup详解"
date: 2020-12-12T15:48:16+08:00
draft: true
---


一、前言
----
    
    Go语言在设计上对同步（Synchronization，数据同步和线程同步）提供大量的支持，比如 goroutine和channel同步原语，库层面有
    
      
    - sync：提供基本的同步原语（比如Mutex、RWMutex、Locker）和 工具类（Once、WaitGroup、Cond、Pool、Map）
    - sync/atomic：提供变量的原子操作（基于硬件指令 compare-and-swap）

-- 引用自[《Golang package sync 剖析(一)： sync.Once》](https://segmentfault.com/a/1190000021650228)

上一期中，我们介绍了 `sync.Once` 如何保障 `exactly once` 语义，本期文章我们介绍 `package sync` 下的另一个工具类：`sync.WaitGroup`。

二、为什么需要 `WaitGroup`？
--------------------

想象一个场景：我们有一个用户画像服务，当一个请求到来时，需要

1.  从 request 里解析出 user_id 和 画像维度参数
2.  根据 user_id 从 ABCDE 五个子服务（数据库服务、存储服务、rpc服务等）拉取不同维度的信息
3.  将读取的信息进行整合，返回给调用方

假设 ABCDE 五个服务的响应时间 p99 是 20~50ms 之间。如果我们顺序调用 ABCDE 读取信息，不考虑数据整合消耗时间，服务端整体响应时间 p99 是：
```go
    sum(A, B, C, D, E) => [100ms, 250ms]
```
先不说业务上能不能接受，响应时间上显然有很大的优化空间。最直观的优化方向就是，取数逻辑的总时间消耗：
```go
    sum(A, B, C, D, E) -> max(A, B, C, D, E)
```
具体到 coding 上，我们需要并行调用 ABCDE 五个子服务，待调用**全部**返回以后，进行数据整合。如何保障`全部`返回呢？

此时，`sync.WaitGroup` 闪耀登场。

三、`WaitGroup` 用法
----------------

官方文档对 WaitGroup 的描述是：`一个 WaitGroup 对象可以等待一组协程结束`。使用方法是：

1.  main协程通过调用 `wg.Add(delta int)` 设置worker协程的个数，然后创建worker协程；
2.  worker协程执行结束以后，都要调用 `wg.Done()`；
3.  main协程调用 `wg.Wait()` 且被block，直到所有worker协程全部执行结束后返回。

这里先看一个典型的例子：
```go
    // src/cmd/compile/internal/ssa/gen/main.go
    func  main() {
      // 省略部分代码 ...
      var wg sync.WaitGroup
      for _, task := range tasks {
        task  := task
        wg.Add(1)
        go func() {
          task()
          wg.Done()
        }()
      }
      wg.Wait()
      // 省略部分代码...
    }
```
这个例子具备了 `WaitGroup` 正确使用的大部分要素，包括：

1.  `wg.Done` 必须在**所有** `wg.Add` 之后执行，所以要保证两个函数都在main协程中调用；
2.  `wg.Done` 在 worker协程里调用，尤其要保证调用一次，不能因为 panic 或任何原因导致没有执行（建议使用 `defer wg.Done()`）；
3.  `wg.Done` 和 `wg.Wait` 在时序上是没有先后。

细心的朋友可能会发现一行非常诡异的代码：
```go
    task  := task
```
Go 对 array/slice 进行遍历时，runtime 会把 `task[i]` 拷贝到 `task` 的内存地址，下标 `i` 会变，而 `task` 的内存地址不会变。如果不进行这次赋值操作，所有 goroutine 可能读到的都是最后一个task。为了让大家有一个直观的感觉，我们用下面这段代码做实验：
```go
    package main
    
    import (
      "fmt"
      "unsafe"
    )
    
    func main() {
      tasks := []func(){
        func() { fmt.Printf("1. ") },
        func() { fmt.Printf("2. ") },
      }
    
      for idx, task := range tasks {
        task()
        fmt.Printf("遍历 = %v, ", unsafe.Pointer(&task))
        fmt.Printf("下标 = %v, ", unsafe.Pointer(&tasks[idx]))
        task  := task
        fmt.Printf("局部变量 = %vn", unsafe.Pointer(&task))
      }
    }
```
这段代码的打印结果是：
```
    1. 遍历 = 0x40c140, 下标 = 0x40c138, 局部变量 = 0x40c150
    2. 遍历 = 0x40c140, 下标 = 0x40c13c, 局部变量 = 0x40c158
```
不同机器上执行打印结果有所不同，但共同点是：

1.  遍历时，数据的内存地址不变
2.  通过下标取数时，内存地址不同
3.  for-loop 内创建的局部变量，即便名字相同，内存地址也不会复用

使用 `WaitGroup` 时，除了上面提到的注意事项，还需要解决数据回收和异常处理的问题。这里我们也提供两种方式供参考：

1.  对于 rpc 调用，可以通过 data channel 和 error channel 搜集信息，或者二合一的channel
2.  共享变量，比如加锁的 map

四、`WaitGroup` 实现
----------------

在讨论这个主题之前，建议读者先思考一下：如果让你去实现 `WaitGroup`，你会怎么做？

锁？肯定不行！

信号量？怎么实现？

------------切入正题------------

在 Go 源码里，`WaitGroup` 在逻辑上包含：

1.  worker 计数器：main协程调用 `wg.Add(delta int)` 时增加 `delta`，调用 `wg.Done`时减一。
2.  waiter 计数器：调用 `wg.Wait` 时，计数器加一; **worker计数器降低到0时，重置waiter计数器**。
3.  信号量：用于阻塞 main协程。调用 `wg.Wait` 时，通过 `runtime_Semacquire` 获取信号量；降低 waiter 计数器时，通过 `runtime_Semrelease` 释放信号量。

为了便于演示，我们魔改一下上面的例子：
```go
    package main
    
    import (
      "fmt"
      "sync"
      "time"
    )
    
    func main() {
      tasks  := []func(){
        func() { time.Sleep(time.Second); fmt.Println("1 sec later") },
        func() { time.Sleep(time.Second *  2); fmt.Println("2 sec later") },
    }
    
      var wg sync.WaitGroup // 1-1
      wg.Add(len(tasks))    // 1-2
      for _, task := range tasks {
        task  := task
        go func() {       // 1-3-1
          defer wg.Done() // 1-3-2
          task()          // 1-3-3
        }()               // 1-3-1
      }
      wg.Wait()           // 1-4
      fmt.Println("exit")
    }
```
上面这段代码中，

1.  1-1 创建一个 `WaitGroup` 对象，worker计数器和waiter计数器默认值均为0。
2.  1-2 设置 worker计数器为 `len(tasks)`。
3.  1-3-1 创建 worker协程，并启动任务。
4.  1-4 设置 waiter计数器，获取信号量，main协程被阻塞。
5.  1-3-3 中执行结束后，1-3-2 降低worker计数器。当worker计数器降低到0时，
    
    *   重置 waiter计数器
    *   释放信号量，main 协程被激活，1-4 `wg.Wait` 返回

尽管 `Add(delta int)` 里 delta 可以是正数、0、负数。我们在使用时，`delta` 总是正数。

`wg.Done` 等价于 `wg.Add(-1)`。在本文中，我们提到 `wg.Add`时，默认 `delta > 0`。

了解了 `WaitGroup` 的原理以后，我们看下它的源码。为了便于理解，我只保留了核心逻辑。对于这部分逻辑，我们分三部分讲解：

1.  `WaitGroup` 结构
2.  `Add` 和 `Done`
3.  `Wait`

**提示：如果只想了解 WaitGroup 的正确用法，本文读到这儿就足够了。对底层有兴趣的朋友可以继续读，不过最好打开IDE，参考源码一起读。**

### 4.1 WaitGroup 结构
```go
    type WaitGroup struct {
      noCopy noCopy
      state1 [3]uint32
    }
```
`WaitGroup` 结构体里有 `noCopy` 和 `state1` 两个字段。

编译代码时，`go vet` 工具会检查 `noCopy` 字段，避免 `WaitGroup` 对象被拷贝。

`state1` 字段比较秀，在逻辑上它包含了 worker计数器、waiter计数器和信号量。具体如何读这三个变量，参考下面代码：
```go
    // state returns pointers to the state and sema fields stored within wg.state1.
    func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
      if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
        return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
      } else {
        return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
      }
    }
    
    // 读取计数器和信号量
    statep, semap := wg.state()
    state  := atomic.LoadUint64(statep)
    v := int32(state >> 32)
    w := uint32(state)
```
三个变量的取数逻辑是：

*   worker计数器：`v` 是 `statep *uint64` 的`左32位`
*   waiter计数器：`w` 是 `statep *uint64` 的`右32位`
*   信号量：`semap` 是 `state1 [3]uint32` 的第一个字节/最后一个字节

所以，更新worker计数器，需要这样做：
```go
    state := atomic.AddUint64(statep, uint64(delta)<<32)
```
更新waiter计数器，需要这样做：
```go
    statep, semap := wg.state()
    for {
      state := atomic.LoadUint64(statep)
      if atomic.CompareAndSwapUint64(statep, state, state+1)   {
        // 忽略其他逻辑
        return
      }
    }
```
细心的朋友可能会发现，worker计数器的更新是直接累加，而 waiter计数器的更新是 CompareAndSwap。这是因为在 main协程中执行 `wg.Add` 时，只有main协程对 `state1` 做修改；而 `wg.Wait` 中修改waiter计数器时，可能有很多个协程在更新 `state1`。如果你还不太理解这段话，不妨先往下走，了解 `wg.Add` 和 `wg.Wait` 的细节之后再回头看。

### 4.2 Add 和 Done

`wg.Add` 操作的核心逻辑比较简单，即修改 worker计数器，根据worker计数器的状态进行后续操作。简化版的代码如下：
```go
    func (wg *WaitGroup) Add(delta int) {
      statep, semap := wg.state()
      // 1. 修改worker计数器
      state := atomic.AddUint64(statep, uint64(delta)<<32)
      v := int32(state >> 32)
      w := uint32(state)
      if v <  0 {
        panic("sync: negative WaitGroup counter")
      }
      if w != 0 && delta > 0 && v == int32(delta) {
        panic("sync: WaitGroup misuse: Add called concurrently with Wait")
      }
      // 2. 判断计数器
      if v > 0 || w == 0 {
        return
      }
      
      // 3. 当 worker计数器降低到0时
      // 重置 waiter计数器，并释放信号量
      *statep = 0
      for ; w != 0; w-- {
        runtime_Semrelease(semap, false)
      }
    }
    
    func (wg *WaitGroup) Done() {
      wg.Add(-1)
    }
```
### 4.3 Wait

`wg.Wait` 的逻辑是修改waiter计数器，并等待信号量被释放。简化版的代码如下：
```go
    func (wg *WaitGroup) Wait() {
      statep, semap  := wg.state()
      for {
        // 1. 读取计数器
        state := atomic.LoadUint64(statep)
        v := int32(state >> 32)
        w := uint32(state)
        if v == 0 {
          return
        }
    
        // 2. 增加waiter计数器
        if atomic.CompareAndSwapUint64(statep, state, state+1) {
          // 3. 获取信号量
          runtime_Semacquire(semap)
          if *statep != 0 {
            panic("sync: WaitGroup is reused before previous Wait has returned")
          }
        
          // 4. 信号量获取成功
          return
        }
      }
    }
```
由于源码比较长，包含了很多校验逻辑和注释，本文中在引用时，在保留核心逻辑的同时均做了不同程度的删减。最后，推荐各位把源码下载下来，细细研读一番，从细节上对 `WaitGroup` 的设计有更深入的理解。
