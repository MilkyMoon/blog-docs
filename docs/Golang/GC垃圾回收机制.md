# Golang的垃圾回收机制
<br desc/>

## 标记清除

标记清除执行过程可以分为两个阶段：
1. 标记：从根对象出发，递归查找并标记所有存活的对象；
2. 清除：遍历所有堆对象，回收未被标记的对象并将回收的内容加入空闲链表；

弊端：垃圾收集过程中程序不能执行

<br desc/>

## 三色标记法
垃圾收集器开始工作时，程序中不存在任何黑色对象，根对象将被标记为灰色。三色标记的步骤可分为以下几个步骤：
1. 从灰色对象集合中选取一个将其标记为黑色；
2. 将黑色对象指向的所有对象标记为灰色；
3. 重复上述两个步骤，直到不存在灰色对象为止；

不希望出现的情况：
1. 一个白色对象被一个黑色对象引用
2. 从灰色对象出发，到达白色对象的、未经访问过的路径被破坏

## 强弱三色不变性
* 强三色不变性 ：黑色对象不会指向白色对象，只会指向灰色对象或者黑色对象；
* 弱三色不变性 ：黑色对象指向的白色对象必须包含一条从灰色对象经由多个白色对象的可达路径;

## 屏障机制
* 插入写屏障：当一个对象被引用的时候触发插入写屏障
* 删除写屏障：当一个对象被删除引用的时候触发删除写屏障

### 插入写屏障
```go
writePointer(slot, ptr):
    shade(ptr)
    *slot = ptr
```
当对象A引用对象B时，将对象B标记为灰色（将在gc结束后被重新扫描）
![2020-03-16-15843705141840-dijkstra-insert-write-barrier](http://img.codwiki.cn/20210604/2020-03-16-15843705141840-dijkstra-insert-write-barrier.png)
标记过程：

1. 将对象A所指向的对象B标记为灰色
2. 用户程序修改对象A的指针，将指向对象B的指针修改为指向对象C的指针，这时触发插入写屏障，将对象C标记为灰色
3. 继续遍历其他灰色对象，将其标记为黑色

栈内存将不执行插入写屏障，所以标记结束时需要进行STW来扫描一遍栈。

### 删除写屏障
```go
writePointer(slot, ptr)
    shade(*slot)
    *slot = ptr
```
当对象A删除引用的对象B时，将对象B标记为灰色（如果删除的是最后一个指向他的指针，则将在下一轮GC中被删除）
![2021-01-02-16095599123266-yuasa-delete-write-barrier](http://img.codwiki.cn/20210604/2021-01-02-16095599123266-yuasa-delete-write-barrier.png)
标记过程：

1. 将对象A所指向的对象B标记为灰色
2.  用户程序修改对象A的指针，将指向对象B的指针修改为指向对象C的指针，指向B的链路被删除，这时触发删除写屏障，因为对象B已经是灰色，所以不做操作
3. 用户程序修改对象B的指针，将指向对象C的指针删除，这时触发删除写屏障，将对象C修改为灰色
4. 继续遍历其他灰色对象，将其标记为黑色

对象B将在下一轮GC中被删除

### 混合写屏障
特性：
1. GC开始将栈上的对象全部标记为黑色
2. GC期间任何栈上创建的对象均为黑色
3. 被删除的对象标记为灰色
4. 被添加的对象标记为灰色

```go
writePointer(slot, ptr):
    shade(*slot)
    if current stack is grey:
        shade(ptr)
    *slot = ptr
```


## GC流程
![1DDB1056-396B-4288-A662-63CED23CDF69](http://img.codwiki.cn/20210604/1DDB1056-396B-4288-A662-63CED23CDF69.png)
关键全局变量：
gcphase：GC工作阶段
writeBarrier.enabled: 是否开启写屏障
gcBlackenEnabled: 是否允许GC标记工作

### 第一阶段：GC开始，准备工作，stop the world
初始所有内存都是白色的，进入标记队列就是灰色
1. STW（Stop To world）
2. 为每个P创建一个mark work 协程（会很快进入休眠状态）
3. 找到所有根对象(stack,heap,global vars)并加入标记队列


### 第二阶段：marking标记阶段，start the world
当所有准备工作做好之后start the world，后台mark work开始调度执行展开标记工作，这个阶段与用户程序是并发执行的。
1. 将状态切换至`GCMark`(gcphase = _GCMark)
2. 开启写屏障(writeBarrier.enabled = true)
3. 允许GC开始标记工作(gcBlackenEnabled=1)
4. 从标记队列中取出对象，标记为黑色
5. 检测是否有指向另一个对象，有则加入标记队列（所有进入队列中的对象逻辑上就认为是灰色的）
6. 扫描过程中如果代码修改了对象，则触发写屏障，将对象标记为灰色，并单独加入到扫描队列中

Golang中分配对象会根据是否是指针分别放到不同的span中，根据这个如果span是指针span，那么就需要继续scan下一个对象，否则停止该路scan，取队列中下一个对象继续scan
![4B310C40-3576-423F-AB57-3A36D79D53CC](http://img.codwiki.cn/20210604/4B310C40-3576-423F-AB57-3A36D79D53CC.png)



堆内存结构
![3FC79FE1-E2E4-44CC-894D-18811677EBE9](http://img.codwiki.cn/20210604/3FC79FE1-E2E4-44CC-894D-18811677EBE9.png)

### 第三阶段：处理marking过程中修改的指针，stop the world
Gc write barrier记录的所有修改的指针也加入标记队列进行一轮标记。确认标记已经完成，停止标记工作。
1. 将状态切换至`_GCMarkTermination `(gcphase = _GCMarkTermination)
2. 关闭GC标记工作(gcBlackenEnabled=0)

### 第四阶段：sweep，start the word
当前阶段内存要么是黑色，要么是白色，将所有白色清除即可。
1. 将状态切换至`_GCOff `(gcphase =_GCOff)
2. 关闭写屏障(writeBarrier.enabled = false)
3. 将sweeper协程加入到run queue中
4. 调度执行sweeper，进行清扫任务

在GCOff之前新分配的对象将被标记为黑色，之后分配的对象被标记为白色

## GC触发条件
1. 内存大小阈值， 内存达到上次gc后的2倍
2. 达到定时时间 ，2m

## 相关文章
[golang gc 简明过程（基于go 1.14） - 知乎](https://zhuanlan.zhihu.com/p/92210761)
 [Go: How Does the Garbage Collector Mark the Memory? | by Vincent Blanchon | A Journey With Go | Medium](https://medium.com/a-journey-with-go/go-how-does-the-garbage-collector-mark-the-memory-72cfc12c6976) 
 [Go 语言垃圾收集器的实现原理 | Go 语言设计与实现 (draveness.me)](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-garbage-collector/) 