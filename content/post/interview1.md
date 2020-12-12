---
title: "【面试】公司1（中小型）"
date: 2020-12-12T16:21:26+08:00
draft: true
---

* **笔试**
1. 详细描述一下web的session；
2. 优化一段代码
    ```go
    func Operator() error {
     if Operator1() {
        if Operator2() {
            if Operator3() {
                if Operator4() {
                    } else {
                        return OPERATORFAIL4
                    }
                } else {
                    return OPERATORFAIL3
                }
             } else {
                return OPERATORFAIL2
             }
         } else {
            return OPERATORFAIL1
         }
     return nil
    }
    ```
3. nil、closed、只读、只写的channel，分别进行write和read会有什么情况发生？
4. 如何避免3的情况发生？我们在使用channel的时候应该注意什么？
5. 给定一大堆ip范围、国家的映射数据文件，根据ip获取国家，如下结构
    ```
    1.2.0.12-1.2.1.123,CN;
    2.23.0.142-2.23.1.123,SG;
    ......
    23.230.0.142-23.230.1.123,US;
    ```
    有以下3个小题:
    * 实现一个函数,把ip转换为一种可比较的数，并且跟ip一样是保持原来的大小排序；

    * 根据上面实现的函数，如何设计数据库表以及设计sql，如何快速的以ip查到国家；
    * 忘了；
6. 给定一组整数，输出指定和为某个数的多对结果；
7. 以下的函数输出
    ```go
    func main() {
         for i := 0; i < 10; i++ {
         go func() {
                fmt.Println(i)
            }() 
         } 
         select {}
    }
    ```
8. 如何实现只用2个goroutine, 打印5个随机数；
* **一面**
    * 项目介绍
    * 针对笔试题进行提问
    * Prometheus的一些问题
    * 消息队列相关问题
* **二面**
    * 项目介绍
    * 围绕项目的一些解决方案（感觉是套方案的。。。）
* **总结**
    * 总体技术面试难度一般，工作环境还可以，技术氛围估计也还ok，不过好像是大小周
    * 欢迎留言回答及纠错