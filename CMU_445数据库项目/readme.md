# CMU 445数据库系统项目
**为什么做**:

- 在结束完MIT的6.824项目, 实现了所谓的分布式分片数据库后, 笔者发现**整个项目完全不考虑数据存储层**, 即所有的实现是一个完备的分布式的**逻辑**数据库. 为更深入认识"一条SQL语句执行的全过程", 特别是在存储引擎层会发生的事, 开始了这个项目.
- 与MySQL的学习同步进行.
- 学习适用于C/C++生产环境的完整工具链 (特别是静态分析工具与测试框架.).


**做了什么**
- project#1: 实现服务DBMS的缓存池 (buffer pool). 实现使用**可扩展哈希** (extendable hashing) 维护数据库页 (page) 和缓存池帧 (frame) 的映射. 实现使用**K-LRU**替换算法将过时的帧替换刷入磁盘的数据库页.
- project#2: 实现支持并发插入/删除的B+树 (插入操作, 删除操作, **插入删除混合**操作均支持并发.). 实现服务**遍历查询**操作的迭代器.

**最终效果**
- project#1: 目前leader board排名464.
- project#2: 目前leader board排名398.
![project 1截图](./images/project1.jpg)
![project 2截图](./images/project2.jpg)
(目前仅进行基础优化, 打榜数据一般.)

## 并发B+树设计思想

Q: **第一次插入操作**因为需要做一些初始化操作有点特殊. 怎么设计支持并发?
> A: 并发的B+树正常情况下需要从根开始向下加锁, 而第一次插入所依赖的根节点还没有建立, 需要做相关的初始化, 一个简单的思路是用`bool is_empty_`维护该信息, 用一个mutex `mux_`保护它. 每次插入执行类似的逻辑:
```
auto insert() -> bool {
    bool is_empty = false;
    mux_.lock();
    is_empty = is_empty_;
    mux_.unlock();
    // ... 其余逻辑

    if (is_empty) {
        // ... 第一次插入的额外初始化逻辑
    }
    // ... 正常插入都需要走的逻辑
}
```
> 它的不足之处在于初始化操作发生的频率是非常低的, 而我们这样必须要从最悲观的角度出发, 每次插入都用一个mutex去获取成员变量`is_empty_`的最新值. **更重要的**, 线程A的insert进入了`if (is_empty)`分支, 如何抑制同时想执行insert的线程B也进入? 线程A何时置`is_empty_`?
因此目前实现使用**double-check locking**. 具体的, `is_empty`使用原子变量`std::atomic<bool>`保证读写原子.

```
private:
  std::atomic<bool> is_empty_;
  std::mutex mux_;
// ...
auto BPLUSTREE_TYPE::IsEmpty() const -> bool { return is_empty_.load(); }

auto insert() -> bool {
if (IsEmpty()) {
    std::lock_guard<std::mutex> lock(mux_);
    if (IsEmpty()) { 
      /*
        初始化逻辑. 主要为向buffer pool申请页, 向本页写入 (本次call提供的) 第一个KV.
        更新root page id供非初始化的操作使用.
      */
      is_empty_.store(false);
      return true;
    }
  }
}
```
> 这样所有尝试初始化的insert操作 (称insert^操作) 突破了外层`if`, 但因为mutex的scope是整个外层`if`, 只有一个这样的insert操作 (称insert_) 突破了内层`if`, 而其他的insert^会被mutex阻挡, 待insert_释放了mutex后均无法突破内层`if`. **而所有不尝试初始化的insert操作不会进入外层`if`**, 也就规避了 (绝大多数情况不需要的) 互斥锁.

Q: 那第一次插入操作和根有关. 有其他和根有关的并发问题嘛?
>A: OK, 根不光会「从无到有」, 还会发生其他形式的变更. 当访问根需要上写锁时, 如果没能成功获取到锁, 我们得想好处理措施. 具体地说, 考虑下面这个目前所有叶子均满了的2阶B+树 (内部/叶子节点均最多2个后代):
![](./images/phantom_root1.png)
此时线程A insert 11, 线程B insert 2发生对root page#6的竞争, 设线程A成功了, 线程B势必要等待, 但然后呢? **等待谁?** 显然新的根不再是page#6.
![](./images/phantom_root2.png)
因此线程B对根的上锁等待, 应是一种「等待检查」的机制. 因此目前实现使用**条件变量**处理「根锁」是否被释放这一共享信息. 获得「根锁」的线程在条件变量的保护下置flag; 在可以释放「根锁」时清flag同时唤醒. 这里因为都是读锁的竞争因此`signal`而非`broadcast`.

```
// 枚举类定义根状态.
enum class RootLockType : size_t { UN_LOCKED = 0, READ_LOCKED = 1, WRITE_LOCKED = 1 << 1 };
private:
  // 条件变量三元组.
  std::mutex mux_;
  std::atomic<size_t> root_locked_;
  std::condition_variable c_v_;
// ...
auto insert () -> bool {
  // 上面说的初始化判定, 操作.
  do {
    std::unique_lock<std::mutex> lock(mux_);
    while (root_locked_.load() != static_cast<size_t>(RootLockType::UN_LOCKED)) {
      c_v_.wait(lock);
      // 被唤醒就检查.
    }
   
    root_locked_.store(static_cast<size_t>(RootLockType::WRITE_LOCKED));

    // 本线程对根上写锁, 获得指向根页的指针.
    break;
  } while (true);
  // ...复杂的逻辑

  if (root 可以释放写锁) {
    // 可以释放根节点的时候唤醒. 未必在函数的末尾才开始对锁的逐个释放.
    mux_.lock();
    root_locked_.store(static_cast<size_t>(RootLockType::UN_LOCKED));
    c_v_.notify_one();
    mux_.unlock();
  }
}
```
![](./images/phantom_root3.png)

Q: 插入还有什么问题嘛?
> A: 有一个问题会影响到B+树的可扩展性, 即一次insert操作会影响改变至多多少页面? 依旧考虑上面这个例子, 首先插入11, 我们必须从上向下上树高$h$个页的锁且**一个都不能放**. 同时这$h$个页面都会分裂还会有一个新的根, 这里就有$2h+1$个.
但早期实现忽略了一点: 当内部节点 (page#A分裂出right sibling page#B) 发生分裂时, page#A之前的孩子页有相当一部分需要更新其parent id! 当B+树**阶数比较大**而buffer pool的大小比较小就容易出问题: 本次insert**实际影响的页数**远远大于$2h+1$甚至大于整个buffer pool的大小.
![](./images/phantom_root1.png)
解决方法是识别页面修改的生命周期. 即考虑page#4, 5这样需要改变parent id的页. 这样的页在最后修改完parent id后, **原则上就从buffer pool被替换然后刷入磁盘**. 即哪怕本次insert**实际影响的页数**非常大, 但需要同时在buffer pool的页数未必很大. 即在插入引发页面分裂时 (准确地说是叶子节点分裂引发了内部节点的级联分裂), 需要及时释放这类**横向的、本次insert仅修改一次的、可能海量的**页.
![](./images/phantom_root2.png)

Q: 提到了**及时释放**这类横向的、本次insert仅修改一次的、可能海量的页.「释放」是什么意思? 遇到过什么坑嘛?
> A: 数据库页因为要被某些线程以`insert,remove,getvalue`语义访问, 形成线程对页面引用的引用计数. 具体来说一个buffer pool的frame#1承载了数据库页的page#a后可能被若干个线程同时上锁, 每个施加一些引用计数. 当引用计数归零时, buffer pool认为没有活跃的线程持续需要它, 于是根据替换算法可以将它刷入磁盘, frame#1可以承载其他数据库页了. **及时释放的含义是**, 在满足正确的前提下, 尽早让页的引用计数归零 (从而承载它的frame能载入新的数据库页).
![](./images/unpin.png)
> 这里有一个坑是: 线程对页面`unlatch`和`unpin` (即让本线程对它的引用计数--) 的顺序. 