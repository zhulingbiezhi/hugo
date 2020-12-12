---
title: "【golang】pprof内存指标"
date: 2020-12-12T16:22:20+08:00
draft: true
---

最近组内一些Go服务碰到内存相关的问题，所以今天抽时间看了下Go pprof内存指标的含义，为后续查问题做准备。  
内容主要来自于Go代码中对这些字段的注释，加自己的理解。理解不对的地方欢迎指正。
```go
// https://github.com/golang/go/blob/master/src/runtime/mstats.go#L150  
  
// 总共从OS申请的字节数  
// 是下面各种XxxSys指标的总和。包含运行时的heap、stack和其他内部数据结构的总和。  
// 它是虚拟内存空间。不一定全部映射成了物理内存。  
Sys  
  
// 见\`Sys\`  
HeapSys  
  
// 还在使用的对象，以及不使用还没被GC释放的对象的字节数  
// 平时应该平缓，gc时可能出现锯齿  
HeapAlloc  
  
// 正在使用的对象字节数。  
// 有个细节是，如果一个span中可包含多个object，只要一个object在使用，那么算的是整个span。  
// \`HeapInuse\` - \`HeapAlloc\`是GC中保留，可以快速被使用的内存。  
HeapInuse  
  
// 已归还给OS的内存。没被堆再次申请的内存。  
HeapReleased  
  
// 没被使用的span的字节数。  
// 这部分内存可以被归还给OS，并且还包含了\`HeapReleased\`。  
// 可以被再次申请，甚至作为栈内存使用。  
// \`HeapIdle\` - \`HeapReleased\`即GC保留的。  
HeapIdle  
  
/// ---  
  
// 和\`HeapAlloc\`一样  
Alloc  
  
// 累计的\`Alloc\`  
// 累计的意思是随程序启动后一直累加增长，永远不会下降。  
TotalAlloc  
  
// 没什么卵用  
Lookups = 0  
  
// 累计分配的堆对象数  
Mallocs  
  
// 累计释放的堆对象数  
Frees  
  
// 存活的对象数。见\`HeapAlloc\`  
// HeapObjects = \`Mallocs\` - \`Frees\`  
HeapObjects  
  
// ---  
// 下面的XxxInuse中的Inuse的含义，和XxxSys中的Sys的含义，基本和\`HeapInuse\`和\`HeapSys\`是一样的  
// 没有XxxIdle，是因为都包含在\`HeapIdle\`里了  
  
// StackSys基本就等于StackInuse，再加上系统线程级别的栈内存  
Stack = StackInuse / StackSys  
  
// 为MSpan结构体使用的内存  
MSpan = MSpanInuse / MSpanSys  
  
// 为MCache结构体使用的内存  
MCache = MCacheInuse / MCacheSys  
  
// 下面几个都是底层内部数据结构用到的XxxSys的内存统计  
BuckHashSys  
GCSys  
OtherSys  
  
// ---  
// 下面是跟GC相关的  
  
// 下次GC的触发阈值，当HeapAlloc达到这个值就要GC了  
NextGC  
  
// 最近一次GC的unix时间戳  
LastGC  
  
// 每个周期中GC的开始unix时间戳和结束unix时间戳  
// 一个周期可能有0次GC，也可能有多次GC，如果是多次，只记录最后一个  
PauseNs  
PauseEnd  
  
// GC次数  
NumGC  
  
// 应用程序强制GC的次数  
NumForcedGC  
  
// GC总共占用的CPU资源。在0~1之间  
GCCPUFraction  
  
// 没被使用，忽略就好   
DebugGC  
```
查看方式
```go
// 方式一  
import "runtime"  
  
var m runtime.MemStats  
runtime.ReadMemStats(&m)  
   
// 方式二  
import \_ "net/http/pprof"  
import "net/http"  
   
http.ListenAndServe("0.0.0.0:10001", nil)  
// http://127.0.0.1:10001/debug/pprof/heap?debug=1  
```
下面随便找个服务来练手。

Top查看程序的`VIRT`约为2.4G，`RES`约为1.7G。

使用web pprof观察到的指标，可以对应着前文说的含义看看。
```
Sys = 1842916040 ~1.7G  
HeapSys = 1711013888 ~1.6G  
HeapInuse = 1237483520 ~1.18G  
HeapAlloc = 1195472528 ~1.14G  
HeapInuse - HeapAlloc = 40M  
HeapIdle = 473530368 ~451M  
HeapReleased = 61063168 ~58.2M  
HeapIdle - HeapReleased = 393M  
  
Alloc = 1195472528 ~1.14G  
TotalAlloc = 426616278424 ~397.3G  
  
Lookups = 0  
Mallocs = 2907819388 ~29亿对象数  
Frees = 2901808898 ~29亿对象数  
HeapObjects = 6010490 ~601万对象数  
  
Stack = 33390592 / 33390592 ~31.8M / 31.8M  
MSpan = 13542744 / 19906560 ~12.9M / 18.9M  
MCache = 55552 / 65536  
BuckHashSys = 2371870  
GCSys = 69398992  
OtherSys = 6768602  
```
好，后续遇到内存有问题的案例，再来补充。

