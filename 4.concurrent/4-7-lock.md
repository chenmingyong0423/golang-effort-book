# 4.7 并发安全与锁

> 本节源码位置 https://github.com/golang-minibear2333/golang/tree/master/4.concurrent/4.7-lock/

> 并发安全，就是多个并发体在同一段时间内访问同一个共享数据,共享数据能被正确处理。

很多语言的并发编程很容易在同时修改某个变量的时候，因为操作不是原子的，而出现错误计算，比如一个加法运算使用中的变量被修改，而导致计算结果出错，典型的像统计商品库存。

个人建议只要涉及到共享变量统统使用`channel`，因为`channel`源码中使用了互斥锁，它是并发安全的。

我们可以不用，但不可以不了解，手中有粮心中不慌。

## 4.7.1 并发不安全的例子

数组是并发不安全的，在例子开始前我们要知道`append`函数的行为：长度足够在原数组`cap`内追加函数，增加`len`，长度不够则触发扩容，申请新数组`cap`增加一倍，赋值迁移。

所以在这个过程中，仅讨论扩容操作的话可能存在同时申请并赋值的情况，导致漏掉某次扩容增加的数据。

```go
var s []int

func appendValue(i int) {
	s = append(s, i)
}

func main() {
	for i := 0; i < 10000; i++ { //10000个协程同时添加切片
		go appendValue(i)
	}
    time.Sleep(2)
    fmt.Println(len(s))
}
```

比如上例，`10000` 个协程同时为切片增加数据，你可以尝试运行一下，打印出来的一定不是 `10000` 。

* 以上并发操作的同一个资源，专业名词叫做**临界区**。
* 因为并发操作存在数据竞争，导致数据值意外改写，最后的结果与期待的不符，这种问题统称为**竞态问题**。

常见于控制商品减库存，控制余额增减等情况。 那么有什么办法解决竞态问题呢？

* 互斥锁：让访问某个临界区的时候，只有一个 `goroutine` 可以访问。
* 原子操作：让某些操作变成原子的，这个后续讨论。

这两个思路贯穿了无数的高并发/分布式方案，区别是在一个进程应用中使用，还是借助某些第三方手段来实现，比如中间件。独孤九剑森罗万象一定要牢牢记住。

## 4.7.2 互斥锁

`Go` 语言中互斥锁的用法如下：

```go
var lock sync.Mutex //互斥锁
lock.Lock() //加锁
s = append(s, i)
lock.Unlock() //解锁
```

在访问临界区的前后加上互斥锁，就可以保证不会出现并发问题。

我们修改还是上一个`4.7.1`的例子，为其增加互斥锁。

```go
var s []int
var lock sync.Mutex

appendValueSafe := func(i int) {
    lock.Lock()
    s = append(s, i)
    lock.Unlock()
}

for i := 0; i < 10000; i++ { //10000个协程同时添加切片
    go appendValueSafe(i)
}
time.Sleep(2)
fmt.Println(len(s))
```

* 对共享变量`s`的写入操作加互斥锁，保证同一时刻只有一个`goroutine`修改内容。
* 加锁之后到解锁之前的内容，同一时刻只有一个访问，无论读写。
* 无论多少次都输出`10000`，不会再出现竞态问题。
* 要注意：**如果在写的同时，有并发读操作时，为了防止不要读取到写了一半数据，需要为读操作也加锁。**

## 4.7.3 读写锁

互斥锁是完全互斥的，并发读没有修改的情况下是不会有问题的，也没有必要在并发读的时候加锁不然效率会变低。

用法：

```go
rwlock sync.RWMutex
//读锁
rwlock.RLock()
rwlock.RUnlock()

//写锁
rwlock.Lock()
rwlock.Unlock()
```

并发读不互斥可以同时，在一个写锁获取时，其他所有锁都等待， 口诀：读读不互斥、读写互斥、写写互斥。具体测算速度的代码可以见4.7.3的源码，感兴趣的可以改下注释位置看下效率是有很明显的提升的。


## 4.7.4 小结

* 学习了几个名词：临界区、竞态问题、互斥锁、原子操作、读写锁。
* 互斥锁：`sync.Mutex`, 读写锁：`sync.RWMutex` 都是 `sync` 包的。
* 读写锁比互斥锁效率高。

问题：只加写锁可以吗？为什么？

## 引用

* [并发安全和锁](https://www.topgoer.com/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/%E5%B9%B6%E5%8F%91%E5%AE%89%E5%85%A8%E5%92%8C%E9%94%81.html)
* [关于go的并发安全](https://zhuanlan.zhihu.com/p/511128412)
* [golang_并发安全: slice和map并发不安全及解决方法](https://blog.csdn.net/weixin_43851310/article/details/87897247)

