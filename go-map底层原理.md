---
title: go 数组和切片
date: 2022-01-24 16:50:31
tags:
---



**map** 其实可以理解为一种**哈希表**数据结构。几乎所有的编程语言都会有数组和哈希表这两种数据结构，有的编程语言将数组称为列表，比如：Python，而有的语言将 map 称作字典或者映射。无论如何命名或者如何实现，数组和哈希是两种设计集合元素的思路，数组用于表示元素的序列，而哈希表示的是键值对之间映射关系。

以下哈希表即为**Map**。

## 设计原理

哈希表是计算机中最重要的数据结构之一，不仅是因为它 𝑂(1) 的时间复杂度，而且读写性能非常好，它还提供了键值之间的映射。想要实现一个性能优异的哈希表，需要注意两个关键点：

-   哈希函数的选择
-   哈希冲突的解决

### 哈希函数

实现哈希表的关键点在于哈希函数的选择，哈希函数的选择在很大程度上能够决定哈希表的读写性能。

实际的方式是让哈希函数的结果能够尽可能的均匀分布，然后通过相应的方法来解决哈希冲突的问题。哈希函数映射的结果一定要尽可能均匀，结果不均匀的哈希函数会带来更多的哈希冲突以及更差的读写性能。

如果使用结果分布较为均匀的哈希函数，那么哈希的增删改查的时间复杂度为 𝑂(1)；但是如果哈希函数的结果分布不均匀，那么所有操作的时间复杂度可能会达到 𝑂(𝑛)，由此看来，选择使用好的哈希函数是至关重要的。

### 冲突解决

前面所提到的，在通常情况下，哈希函数是不一定完美的，当输入的键足够多也会产生冲突，所以仍然存在发生哈希碰撞的可能，这时就需要一些方法来解决哈希碰撞的问题，常见方法的有：**开放寻址法**和**拉链法**。

#### 开放寻址法

**开放寻址法**是一种在哈希表中解决哈希碰撞的比较普通的方法，这种方法的核心思想是：**依次探测和比较数组中的元素以判断目标键值对是否存在于哈希表中**，如果我们使用开放寻址法来实现哈希表，那么实现哈希表底层的数据结构就是数组，不过因为数组的长度有限。

当我们向当前哈希表写入新的数据时，如果发生了冲突，就会将键值对写入到下一个索引不为空的位置，如下：

![开放寻址法.png](https://raw.githubusercontent.com/ycchildcoder/markdown/main/2066f1668cd34ff8b6a477320452c0a7%7Etplv-k3u1fbpfcp-watermark.image)

如上图所示，当 Key3 与已经存入哈希表中的两个键值对 Key1 和 Key2 发生冲突时，Key3 会被写入 Key2 后面的空闲位置。当我们再去读取 Key3 对应的值时就会先获取键的哈希并取模，这会先帮助我们找到 Key1，找到 Key1 后发现它与 Key 3 不相等，所以会继续查找后面的元素，直到内存为空或者找到目标元素。

![开放寻址法2.png](https://raw.githubusercontent.com/ycchildcoder/markdown/main/78c696b156104081a3ffbdd43a1584f7%7Etplv-k3u1fbpfcp-watermark.image)

开放寻址法中对性能影响最大的是**装载因子**，它是数组中元素的数量与数组大小的比值。随着装载因子的增加，线性探测的平均用时就会逐渐增加，这会影响哈希表的读写性能。当装载率超过 70% 之后，哈希表的性能就会急剧下降，而一旦装载率达到 100%，整个哈希表就会完全失效，这时查找和插入任意元素的时间复杂度都是 𝑂(𝑛) 的，这时需要遍历数组中的全部元素，所以在实现哈希表时一定要关注装载因子的变化。

#### 拉链法

与开放地址法相比，**拉链法是哈希表最常见的实现方法**，大多数的编程语言都用拉链法实现哈希表，它的实现比较开放地址法稍微复杂一些，但是平均查找的长度也比较短，各个用于存储节点的内存都是动态申请的，可以节省比较多的存储空间。

实现拉链法一般会使用数组加上链表，不过一些编程语言会在拉链法的哈希中引入红黑树以优化性能，拉链法会使用链表数组作为哈希底层的数据结构，我们可以将它看成可以扩展的二维数组：

![拉链法.png](https://raw.githubusercontent.com/ycchildcoder/markdown/main/f3dd3a752779488c9a6a74c3525407be%7Etplv-k3u1fbpfcp-watermark.image)

如上图所示，当我们需要将一个键值对 (Key6, Value6) 写入哈希表时，键值对中的键 Key6 都会先经过一个哈希函数，哈希函数返回的哈希会帮助我们选择一个桶，和开放地址法一样，选择桶的方式是直接对哈希返回的结果取模：

```go
index := hash("Key6") % array.len
复制代码
```

选择了 2 号桶后就可以遍历当前桶中的链表了，在遍历链表的过程中会遇到以下两种情况：

1.  找到键相同的键值对 — 更新键对应的值；
2.  没有找到键相同的键值对 — 在链表的末尾追加新的键值对；



首先**Go 语言采用的是哈希查找表，并且使用链表解决哈希冲突。**

##### GO的内存模型

先看这一张map原理图

<img src="https://raw.githubusercontent.com/ycchildcoder/markdown/main/49dfa7b81e19fbab143ddc0a7b3b7fa0.png" alt="go-map源码简单分析（map遍历为什么时随机的）" style="zoom: 80%;" />

map

再来看看源码中map的定义

```go
//src/runtime/map.go  line 115

// A header for a Go map.
type hmap struct {
    // Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
    // Make sure this stays in sync with the compiler's definition.
    //
    count     int  //len(map)时，返回的值
    flags     uint8   //表示是否有其他协程在操作map
    B         uint8    //上图中[]bmap的''长度'' 2^B
    noverflow uint16  //// 溢出的bucket个数
    hash0     uint32 // hash seed

    buckets    unsafe.Pointer    //buckets 数组指针
    oldbuckets unsafe.Pointer  // 扩容的时候用于赋值的buckets数组
    nevacuate  uintptr       // 搬迁进度

    extra *mapextra   // 用于扩容的指针

    type mapextra struct {
    overflow    *[]*bmap
    oldoverflow *[]*bmap
    nextOverflow *bmap
}

// A bucket for a Go map.
type bmap struct {
    tophash [bucketCnt]uint8        // len为8的数组
}
//底层定义的常量 
const (
    // Maximum number of key/value pairs a bucket can hold.
    bucketCntBits = 3
    bucketCnt     = 1 << bucketCntBits

} 
```



但这只是表面(src/runtime/hashmap.go)的结构，编译期间会给它加料，动态地创建一个新的结构：

```go
type bmap struct {
  topbits  [8]uint8
  keys     [8]keytype
  values   [8]valuetype
  pad      uintptr
  overflow uintptr
} 
```

bmap 就是我们常说的“桶”，**桶里面会最多装 8 个 key，这些 key 之所以会落入同一个桶，是因为它们经过哈希计算后，哈希结果是“一类”的(低位的B位决定bucket)**。在桶内，又会根据 **key 计算出来的 hash 值的高 8 位来决定 key 到底落入桶内的哪个位置**（一个桶内最多有8个位置）。如上图所示

bmap 是存放 k-v 的地方，我们把视角拉近，仔细看 bmap 的内部组成。

<img src="https://raw.githubusercontent.com/ycchildcoder/markdown/main/218617d6128e5e0af684d7cbbd72c81d.png" alt="go-map源码简单分析（map遍历为什么时随机的）" style="zoom: 50%;" />

上图就是 bucket 的内存模型， **HOBHash 指的就是 top hash**。注意到 key 和 value 是各自放在一起的，并不是 key/value/key/value/… 这样的形式。源码里说明**这样的好处是在某些情况下可以省略掉 padding 字段，节省内存空间**。

每个 bucket 设计成最多只能放 8 个 key-value 对，**如果有第 9 个 key-value 落入当前的 bucket，那就需要再构建一个 bucket ，通过 overflow 指针连接起来**。

### 创建map

从语法上来说，创建一个map很简单（记得**key的类型必须为可比较类型**）

```go
maps := make(map[string]int)
    maps2 := map[string]int{
        "1":1,
        "2":2,
        "3":3,
    }
    var maps3 map[string]int 
```

通过汇编语言可以看到，实际上底层调用的是 makemap 函数，主要做的工作就是初始化 hmap 结构体的各种字段，例如计算 B 的大小，设置哈希种子 hash0 等等。

```go
func makemap(t *maptype, hint int64, h *hmap, bucket unsafe.Pointer) *hmap 
```

注意，这个函数返回的结果：*hmap，它是一个指针

```go
func makeslice(et *_type, len, cap int) slice 
```



#### hash函数

map 的一个关键点在于，哈希函数的选择。在程序启动时，会检测 cpu 是否支持 aes，如果支持，则**使用 aes hash，否则使用 memhash**。这是在函数 alginit() 中完成，位于路径：src/runtime/alg.go 下。

>   hash 函数，有加密型和非加密型。加密型的一般用于加密数据、数字摘要等，典型代表就是 md5、sha1、sha256、aes256 这种；非加密型的一般就是查找。在 map 的应用场景中，用的是查找。选择 hash 函数主要考察的是两点：性能、碰撞概率。

#### key的定位过程

key 经过哈希计算后得到哈希值，共 64 个 bit 位（64位机，32位机就不讨论了，现在主流都是64位机），计算它**到底要落在哪个桶时，只会用到最后 B 个 bit 位**。还记得前面提到过的 B 吗？如果 B = 5，那么桶的数量，也就是 buckets 数组的长度是 2^5 = 32。
例如，现在有一个 key 经过哈希函数计算后，得到的哈希结果是：

```
10010111 | 000011110110110010001111001010100010010110010101010 │ 01010 
```

**用最后的 5 个 bit 位，也就是 01010，值为 10，也就是 10 号桶**。这个操作实际上就是取余操作，但是取余开销太大，所以代码实现上用的位操作代替。
**再用哈希值的高 8 位，找到此 key 在 bucket 中的位置**，这是在寻找已有的 key。最开始桶内还没有 key，新加入的 key 会找到第一个空位，放入。

buckets 编号就是桶编号，当两个不同的 key 落在同一个桶中，也就是发生了哈希冲突。**冲突的解决手段是用链表法**：在 bucket 中，从前往后找到第一个空位。这样，在查找某个 key 时，先找到对应的桶，再去遍历 bucket 中的 key。

这里参考一张图

<img src="https://raw.githubusercontent.com/ycchildcoder/markdown/main/89f0a0cee0ed90e4b19b24f2579ab92d.png" alt="go-map源码简单分析（map遍历为什么时随机的）" style="zoom: 67%;" />

#### key定位过程

上图中，假定 B = 5，所以 bucket 总数就是 2^5 = 32。首先计算出待查找 key 的哈希，使用低 5 位 00110，找到对应的 6 号 bucket，使用高 8 位 10010111，对应十进制 151，在 6 号 bucket 中寻找 tophash 值（HOB hash）为 151 的 key，找到了 2 号槽位，这样整个查找过程就结束了。

如果在 bucket 中没找到，并且 overflow 不为空，还要继续去 overflow bucket 中寻找，直到找到或是所有的 key 槽位都找遍了，包括所有的 overflow bucket。

看看key的查找过程

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
  //...
  // 如果 h 什么都没有，返回零值
  if h == nil || h.count == 0 {
    return unsafe.Pointer(&zeroVal[0])
  }
  // 写和读冲突
  if h.flags&hashWriting != 0 {
    throw("concurrent map read and map write")
  }
  // 不同类型 key 使用的 hash 算法在编译期确定
  alg := t.key.alg
  // 计算哈希值，并且加入 hash0 引入随机性
  hash := alg.hash(key, uintptr(h.hash0))
  // 比如 B=5，那 m 就是31，二进制是全 1
  // 求 bucket num 时，将 hash 与 m 相与，
  // 达到 bucket num 由 hash 的低 8 位决定的效果
  m := bucketMask(h.B)
  // b 就是 bucket 的地址
  b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
  // oldbuckets 不为 nil，说明发生了扩容
  if c := h.oldbuckets; c != nil {
    // 如果不是同 size 扩容（看后面扩容的内容）
    // 对应条件 1 的解决方案
    if !h.sameSizeGrow() {
      // 新 bucket 数量是老的 2 倍
      m >>= 1
    }
    // 求出 key 在老的 map 中的 bucket 位置
    oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
    // 如果 oldb 没有搬迁到新的 bucket
    // 那就在老的 bucket 中寻找
    if !evacuated(oldb) {
      b = oldb
    }
  }
  // 计算出高 8 位的 hash
  // 相当于右移 56 位，只取高8位
  top := tophash(hash)
  //开始寻找key
  for ; b != nil; b = b.overflow(t) {
    // 遍历 8 个 bucket
    for i := uintptr(0); i < bucketCnt; i++ {
      // tophash 不匹配，继续
      if b.tophash[i] != top {
        continue
      }
      // tophash 匹配，定位到 key 的位置
      k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
      // key 是指针
      if t.indirectkey {
        // 解引用
        k = *((*unsafe.Pointer)(k))
      }
      // 如果 key 相等
      if alg.equal(key, k) {
        // 定位到 value 的位置
        v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
        // value 解引用
        if t.indirectvalue {
          v = *((*unsafe.Pointer)(v))
        }
        return v
      }
    }
  }
  return unsafe.Pointer(&zeroVal[0])
} 
```

函数返回 h[key] 的指针，如果 h 中没有此 key，那就会返回一个 key 相应类型的零值，不会返回 nil。

bucket 里 key 的起始地址就是 unsafe.Pointer(b)+dataOffset。第 i 个 key 的地址就要在此基础上跨过 i 个 key 的大小；而我们又知道，value 的地址是在所有 key 之后，因此第 i 个 value 的地址还需要加上所有 key 的偏移。

```go
// key 定位公式
k :=add(unsafe.Pointer(b),dataOffset+i*uintptr(t.keysize))

// value 定位公式
v:= add(unsafe.Pointer(b),dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))

//对于 bmap 起始地址的偏移：
dataOffset = unsafe.Offsetof(struct{
  b bmap
  v int64
}{}.v) 
```

#### map读取的两个操作

Go 语言中读取 map 有两种语法：带 comma 和 不带 comma。当要查询的 key 不在 map 里，带 comma 的用法会返回一个 bool 型变量提示 key 是否在 map 中；而不带 comma 的语句则会返回一个 value 类型的零值。如果 value 是 int 型就会返回 0，如果 value 是 string 类型，就会返回空字符串。

```go
value := maps["1"]
value2,ok := maps["1"] 
```

以前一直觉得好神奇，怎么实现的？这其实是编译器在背后做的工作：分析代码后，将两种语法对应到底层两个不同的函数。

```go
//src/runtime/map.go line 394
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer 
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool) 
```

#### map如何扩容

使用哈希表的目的就是要快速查找到目标 key，然而，随着向 map 中添加的 key 越来越多，key 发生碰撞的概率也越来越大。bucket 中的 8 个 cell 会被逐渐塞满，查找、插入、删除 key 的效率也会越来越低。
Go 语言采用一个 bucket 里装载 8 个 key，定位到某个 bucket 后，还需要再定位到具体的 key，这实际上又用了时间换空间。
所有的 key 都落在了同一个 bucket 里，a直接退化成了链表，各种操作的效率直接降为 O(n)。

因此，需要有一个指标来衡量前面描述的情况，这就是 装载因子。Go 源码里这样定义 装载因子：loadFactor := count /(2^B)

count 就是 map 的元素个数，2^B 表示 bucket 数量。

**再来说触发 map 扩容的时机：在向 map 插入新 key 的时候，会进行条件检测，符合下面这 2 个条件，就会触发扩容：**

1.  **装载因子超过阈值，源码里定义的阈值是 6.5。**
2.  **overflow 的 bucket 数量过多：当 B 小于 15，也就是 bucket 总数 2^B 小于 2^15 时，overflow 的 bucket 数量超过 2^B；扩容**
    **当 B >= 15，也就是 bucket 总数 2^B 大于等于 2^15， overflow 的 bucket 数量超过 2^15。扩容。**

触发扩容的代码如下

```go
// If we hit the max load factor or we have too many overflow buckets,
    // and we're not already in the middle of growing, start growing.
    if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
        hashGrow(t, h)
        
    }

// overLoadFactor reports whether count items placed in 1<<B buckets is over loadFactor.
func overLoadFactor(count int, B uint8) bool {
    return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}


func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
    // If the threshold is too low, we do extraneous work.
    // If the threshold is too high, maps that grow and shrink can hold on to lots of unused memory.
    // "too many" means (approximately) as many overflow buckets as regular buckets.
    // See incrnoverflow for more details.
    if B > 15 {
        B = 15
    }
    // The compiler doesn't see here that B < 16; mask B to generate shorter shift code.
    return noverflow >= uint16(1)<<(B&15)
} 
```



第 1 点：我们知道，每个 bucket 有 8 个空位，在没有溢出，且所有的桶都装满了的情况下，装载因子算出来的结果是 8。因此当装载因子超过 6.5 时，表明很多 bucket 都快要装满了，查找效率和插入效率都变低了。在这个时候进行扩容是有必要的。

第 2 点：是对第 1 点的补充。就是说在装载因子比较小的情况下，这时候 map 的查找和插入效率也很低，而第 1 点识别不出来这种情况。表面现象就是计算装载因子的分子比较小，即 map 里元素总数少，但是 bucket 数量多（真实分配的 bucket 数量多，包括大量的 overflow bucket）。

不难想像造成这种情况的原因：不停地插入、删除元素。先插入很多元素，导致创建了很多 bucket，但是装载因子达不到第 1 点的临界值，未触发扩容来缓解这种情况。之后，删除元素降低元素总数量，再插入很多元素，导致创建很多的 overflow bucket，但就是不会触犯第 1 点的规定，你能拿我怎么办？overflow bucket 数量太多，导致 key 会很分散，查找插入效率低得吓人，因此出台第 2 点规定。这就像是一座空城，房子很多，但是住户很少，都分散了，找起人来很困难。



对于条件 1，元素太多，而 bucket 数量太少，很简单：将 B 加 1，bucket 最大数量（2^B）直接变成原来 bucket 数量的 2 倍。于是，就有新老 bucket 了。注意，这时候元素都在老 bucket 里，还没迁移到新的 bucket 来。而且，新 bucket 只是最大数量变为原来最大数量（2^B）的 2 倍（2^B * 2）。

如果插入 map 的 key 哈希都一样，就会落到同一个 bucket 里，超过 8 个就会产生 overflow bucket，结果也会造成 overflow bucket 数过多。移动元素其实解决不了问题，因为这时整个哈希表已经退化成了一个链表，操作效率变成了 O(n)。

再来看一下扩容具体是怎么做的。由于 map 扩容需要将原有的 key/value 重新搬迁到新的内存地址，如果有大量的 key/value 需要搬迁，会非常影响性能。因此 **Go map 的扩容采取了一种称为“渐进式”地方式**，原有的 key 并不会一次性搬迁完毕，每次最多只会搬迁 2 个 bucket。

hashGrow() 函数实际上并没有真正地“搬迁”，它只是分配好了新的 buckets，并将老的 buckets 挂到了 oldbuckets 字段上。真正搬迁 buckets 的动作在 growWork() 函数中，而调用 growWork() 函数的动作是在 mapassign 和 mapdelete 函数中。也就是插入或修改、删除 key 的时候，都会尝试进行搬迁 buckets 的工作。先检查 oldbuckets 是否搬迁完毕，具体来说就是检查 oldbuckets 是否为 nil。

```go
func hashGrow(t *maptype, h *hmap) {
  // B+1 相当于是原来 2 倍的空间
  bigger := uint8(1)
  // 对应条件 2
  if !overLoadFactor(h.count+1, h.B) {
    // 进行等量的内存扩容，所以 B 不变
    bigger = 0
    h.flags |= sameSizeGrow
  }
  // 将老 buckets 挂到 buckets 上
  oldbuckets := h.buckets
  // 申请新的 buckets 空间
  newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)
    //先把 h.flags 中 iterator 和 oldIterator 对应位清 0
    //如果 iterator 位为 1，把它转接到 oldIterator 位，使得 oldIterator 标志位变成1
    //可以理解为buckets 现在挂到了 oldBuckets 名下了，将对应的标志位也转接过去
  flags := h.flags &^ (iterator | oldIterator)
  if h.flags&iterator != 0 {
    flags |= oldIterator
  }
  // commit the grow (atomic wrt gc)
  h.B += bigger
  h.flags = flags
  h.oldbuckets = oldbuckets
  h.buckets = newbuckets
  // 搬迁进度为 0
  h.nevacuate = 0
  // overflow buckets 数为 0
  h.noverflow = 0
} 
```



主要是申请到了新的 buckets 空间，把相关的标志位都进行了处理：例如标志 nevacuate 被置为 0， 表示当前搬迁进度为 0。
转移的几个标志位如下：

```go
// 可能有迭代器使用 buckets
iterator = 1
// 可能有迭代器使用 oldbuckets
oldIterator = 2
// 有协程正在向 map 中写入 key
hashWriting  =  4
// 等量扩容（对应条件 2）
sameSizeGrow  = 8
// 可能有迭代器使用 buckets
iterator = 1

// 可能有迭代器使用 oldbuckets
oldIterator = 2

// 有协程正在向 map 中写入 key
hashWriting = 4

// 等量扩容（对应条件 2）
sameSizeGrow = 8 
```



再来看看真正执行搬迁工作的 growWork() 函数。

```go
func growWork(t *maptype, h *hmap, bucket uintptr) {
  // 搬迁正在使用的旧 bucket
  evacuate(t, h, bucket&h.oldbucketmask())
  // 再搬迁一个 bucket，以加快搬迁进程
  if h.growing() {
    evacuate(t, h, h.nevacuate)
  }
}

func (h *hmap) growing() bool {
  return h.oldbuckets != nil
} 
```



bucket&h.oldbucketmask() 这行代码，如源码注释里说的，是为了确认搬迁的 bucket 是我们正在使用的 bucket。oldbucketmask() 函数返回扩容前的 map 的 bucketmask。

搬迁过程evacuate源码：

```go
type evacDst struct {
  b *bmap          // 表示bucket 移动的目标地址
  i int            // 指向 x,y 中 key/val 的 index
  k unsafe.Pointer // 指向 x，y 中的 key
  v unsafe.Pointer // 指向 x，y 中的 value
}

func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
  // 定位老的 bucket 地址
  b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
  // 计算容量 结果是 2^B，如 B = 5，结果为32
  newbit := h.noldbuckets()
  // 如果 b 没有被搬迁过
  if !evacuated(b) {
    // 默认是等 size 扩容，前后 bucket 序号不变
    var xy [2]evacDst
    // 使用 x 来进行搬迁
    x := &xy[0]
    x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
    x.k = add(unsafe.Pointer(x.b), dataOffset)
    x.v = add(x.k, bucketCnt*uintptr(t.keysize))

    // 如果不是等 size 扩容，前后 bucket 序号有变
    if !h.sameSizeGrow() {
      // 使用 y 来进行搬迁
      y := &xy[1]
      // y 代表的 bucket 序号增加了 2^B
      y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
      y.k = add(unsafe.Pointer(y.b), dataOffset)
      y.v = add(y.k, bucketCnt*uintptr(t.keysize))
    }
    // 遍历所有的 bucket，包括 overflow buckets b 是老的 bucket 地址
    for ; b != nil; b = b.overflow(t) {
      k := add(unsafe.Pointer(b), dataOffset)
      v := add(k, bucketCnt*uintptr(t.keysize))
      // 遍历 bucket 中的所有 cell
      for i := 0; i < bucketCnt; i, k, v = i+1, add(k, uintptr(t.keysize)), add(v, uintptr(t.valuesize)) {
        // 当前 cell 的 top hash 值
        top := b.tophash[i]
        // 如果 cell 为空，即没有 key
        if top == empty {
          // 那就标志它被"搬迁"过
          b.tophash[i] = evacuatedEmpty
          continue
        }
        // 正常不会出现这种情况
        // 未被搬迁的 cell 只可能是 empty 或是
        // 正常的 top hash（大于 minTopHash）
        if top < minTopHash {
          throw("bad map state")
        }
        // 如果 key 是指针，则解引用
        k2 := k
        if t.indirectkey {
          k2 = *((*unsafe.Pointer)(k2))
        }
        var useY uint8
        // 如果不是等量扩容
        if !h.sameSizeGrow() {
          // 计算 hash 值，和 key 第一次写入时一样
          hash := t.key.alg.hash(k2, uintptr(h.hash0))
          // 如果有协程正在遍历 map 如果出现 相同的 key 值，算出来的 hash 值不同
          if h.flags&iterator != 0 && !t.reflexivekey && !t.key.alg.equal(k2, k2) {
            // useY =1 使用位置Y
            useY = top & 1
            top = tophash(hash)
          } else {
            // 第 B 位置 不是 0
            if hash&newbit != 0 {
              //使用位置Y
              useY = 1
            }
          }
        }

        if evacuatedX+1 != evacuatedY {
          throw("bad evacuatedN")
        }
        //决定key是裂变到 X 还是 Y
        b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
        dst := &xy[useY]                 // evacuation destination
        // 如果 xi 等于 8，说明要溢出了
        if dst.i == bucketCnt {
          // 新建一个 bucket
          dst.b = h.newoverflow(t, dst.b)
          // xi 从 0 开始计数
          dst.i = 0
          //key移动的位置
          dst.k = add(unsafe.Pointer(dst.b), dataOffset)
          //value 移动的位置
          dst.v = add(dst.k, bucketCnt*uintptr(t.keysize))
        }
        // 设置 top hash 值
        dst.b.tophash[dst.i&(bucketCnt-1)] = top // mask dst.i as an optimization, to avoid a bounds check
        // key 是指针
        if t.indirectkey {
          // 将原 key（是指针）复制到新位置
          *(*unsafe.Pointer)(dst.k) = k2 // copy pointer
        } else {
          // 将原 key（是值）复制到新位置
          typedmemmove(t.key, dst.k, k) // copy value
        }
        //value同上
        if t.indirectvalue {
          *(*unsafe.Pointer)(dst.v) = *(*unsafe.Pointer)(v)
        } else {
          typedmemmove(t.elem, dst.v, v)
        }
        // 定位到下一个 cell
        dst.i++
        dst.k = add(dst.k, uintptr(t.keysize))
        dst.v = add(dst.v, uintptr(t.valuesize))
      }
    }
    // Unlink the overflow buckets & clear key/value to help GC.

    // bucket搬迁完毕 如果没有协程在使用老的 buckets，就把老 buckets 清除掉，帮助gc
    if h.flags&oldIterator == 0 && t.bucket.kind&kindNoPointers == 0 {
      b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
      ptr := add(b, dataOffset)
      n := uintptr(t.bucketsize) - dataOffset
      memclrHasPointers(ptr, n)
    }
  }
  // 更新搬迁进度
  if oldbucket == h.nevacuate {
    advanceEvacuationMark(h, t, newbit)
  }
} 
```



通过前面的说明我们知道，应对条件 1，新的 buckets 数量是之前的一倍，应对条件 2，新的 buckets 数量和之前相等。
对于条件 1，从老的 buckets 搬迁到新的 buckets，由于 bucktes 数量不变，因此可以按序号来搬，比如原来在 0 号 bucktes，到新的地方后，仍然放在 0 号 buckets。

对于条件 2，就没这么简单了。要重新计算 key 的哈希，才能决定它到底落在哪个 bucket。例如，原来 B = 5，计算出 key 的哈希后，只用看它的低 5 位，就能决定它落在哪个 bucket。扩容后，B 变成了 6，因此需要多看一位，它的低 6 位决定 key 落在哪个 bucket。这称为 rehash。

![go-map源码简单分析（map遍历为什么时随机的）](https://raw.githubusercontent.com/ycchildcoder/markdown/main/548d2e743708ee02f875a57decddc18e.png)

rehash

因此，某个 key 在搬迁前后 bucket 序号可能和原来相等，也可能是相比原来加上 2^B（原来的 B 值），取决于 hash 值 第 6 bit 位是 0 还是 1。

**理解了上面 bucket 序号的变化，我们就可以回答另一个问题了：为什么遍历 map 是无序的？**

**map 在扩容后，会发生 key 的搬迁，**原来落在同一个 bucket 中的 key，搬迁后，有些 key 就要远走高飞了（bucket 序号加上了 2^B）。而遍历的过程，就是按顺序遍历 bucket，同时按顺序遍历 bucket 中的 key。搬迁后，key 的位置发生了重大的变化，有些 key 飞上高枝，有些 key 则原地不动。这样，遍历 map 的结果就不可能按原来的顺序了。

**按理说每次遍历这样的 map 都会返回一个固定顺序的 key/value 序列吧。的确是这样，但是 Go 杜绝了这种做法，**因为这样会给新手程序员带来误解，以为这是一定会发生的事情，在某些情况下，可能会酿成大错。

Go 做得更绝，当我们在遍历 map 时，并不是固定地从 0 号 bucket 开始遍历，每次都是从一个随机值序号的 bucket 开始遍历，并且是从这个 bucket 的一个随机序号的 cell 开始遍历。这样，即使你是一个写死的 map，仅仅只是遍历它，也不太可能会返回一个固定序列的 key/value 对了

再明确一个问题：如果扩容后，B 增加了 1，意味着 buckets 总数是原来的 2 倍，原来 1 号的桶“裂变”到两个桶。

例如，原始 B = 2，1号 bucket 中有 2 个 key 的哈希值低 3 位分别为：010，110。由于原来 B = 2，所以低 2 位 10 决定它们落在 2 号桶，现在 B 变成 3，所以 010、 110 分别落入 2、6 号桶。(这与数据库的动态hash索引类似)

<img src="https://raw.githubusercontent.com/ycchildcoder/markdown/main/d57b1782ea46c771a236797bce78f9b1.png" alt="go-map源码简单分析（map遍历为什么时随机的）" style="zoom:50%;" />

有一个特殊情况是：有一种 key，每次对它计算 hash，得到的结果都不一样。这个 key 就是 math.NaN() 的结果，它的含义是 nota number，类型是 float64。当它作为 map 的 key，在搬迁的时候，会遇到一个问题：再次计算它的哈希值和它当初插入 map 时的计算出来的哈希值不一样！

你可能想到了，这样带来的一个后果是，这个 key 是永远不会被 Get 操作获取的！当我使用 m[math.NaN()] 语句的时候，是查不出来结果的。这个 key 只有在遍历整个 map 的时候，才有机会现身。所以，可以向一个 map 插入任意数量的 math.NaN() 作为 key。

下面通过图来宏观地看一下扩容前后的变化。

扩容前，B = 2，共有 4 个 buckets，lowbits 表示 hash 值的低位。假设我们不关注其他 buckets 情况，专注在 2 号 bucket。并且假设 overflow 太多，触发了等量扩容（对应于前面的条件 2）。

![go-map源码简单分析（map遍历为什么时随机的）](https://raw.githubusercontent.com/ycchildcoder/markdown/main/c874b78e4e848c734c140bec8c3cfaa2.png)

扩容完成后，overflow bucket 消失了，key 都集中到了一个 bucket，更为紧凑了，提高了查找的效率。

<img src="https://raw.githubusercontent.com/ycchildcoder/markdown/main/be34bd4ef615d97b0591b13f4defdfc2.png" alt="go-map源码简单分析（map遍历为什么时随机的）" style="zoom:80%;" />

假设触发了 2 倍的扩容，那么扩容完成后，老 buckets 中的 key 分裂到了 2 个 新的 bucket。一个在 x part，一个在 y 的 part。依据是 hash 的 lowbits。新 map 中 0-3称为 x part， 4-7 称为 y part。

![go-map源码简单分析（map遍历为什么时随机的）](https://raw.githubusercontent.com/ycchildcoder/markdown/main/f465b1492d831e61559c2bf9a2f47462.png)

### map遍历

本来 map 的遍历过程比较简单：遍历所有的 bucket 以及它后面挂的 overflow bucket，然后挨个遍历 bucket 中的所有 cell。每个 bucket 中包含 8 个 cell，从有 key 的 cell 中取出 key 和 value，这个过程就完成了。

但是，现实并没有这么简单。还记得前面讲过的扩容过程吗？扩容过程不是一个原子的操作，它每次最多只搬运 2 个 bucket，所以如果触发了扩容操作，那么在很长时间里，map 的状态都是处于一个中间态：有些 bucket 已经搬迁到新家，而有些 bucket 还待在老地方。

因此，遍历如果发生在扩容的过程中，就会涉及到遍历新老 bucket 的过程，这是难点所在。

关于 map 迭代，先是调用 mapiterinit 函数初始化迭代器，然后循环调用 mapiternext 函数进行 map 迭代。

![go-map源码简单分析（map遍历为什么时随机的）](https://raw.githubusercontent.com/ycchildcoder/markdown/main/7d6850297eb459093f60af43d8336a08.png)

前面已经提到过，即使是对一个写死的 map 进行遍历，每次出来的结果也是无序的。下面我们就可以近距离地观察他们的实现了。

```go
func mapiterinit(t *maptype, h *hmap, it *hiter) {
  ...
  it.t = t
  it.h = h
  it.B = h.B
  it.buckets = h.buckets
  if t.bucket.kind&kindNoPointers != 0 {
    h.createOverflow()
    it.overflow = h.extra.overflow
    it.oldoverflow = h.extra.oldoverflow
  }

  r := uintptr(fastrand())
  if h.B > 31-bucketCntBits {
    r += uintptr(fastrand()) << 31
  }
  it.startBucket = r & bucketMask(h.B)
  it.offset = uint8(r >> h.B & (bucketCnt - 1))
  it.bucket = it.startBucket
    ...

  mapiternext(it)
} 
```



重点是fastrand 的部分，是一个生成随机数的方法：它生成了随机数。用于决定从哪里开始循环迭代。更具体的话就是根据随机数，选择一个桶位置作为起始点进行遍历迭代因此每次重新 for range map，你见到的结果都是不一样的。那是因为它的起始位置根本就不固定！

```go
 ...
// decide where to start
r := uintptr(fastrand())
if h.B > 31-bucketCntBits {
  r += uintptr(fastrand()) << 31
}
it.startBucket = r & bucketMask(h.B)
it.offset = uint8(r >> h.B & (bucketCnt - 1))

// iterator state
it.bucket = it.startBucket 
```



### map的赋值和更新

向 map 中插入或者修改 key，最终调用的是 mapassign 函数。

**实际上插入或修改 key 的语法是一样的，只不过前者操作的 key 在 map 中不存在，而后者操作的 key 存在 map 中。**

mapassign 有一个系列的函数，根据 key 类型的不同，编译器会将其优化为相应的“快速函数”。

整体来看，流程非常得简单：对 key 计算 hash 值，根据 hash 值按照之前的流程，找到要赋值的位置（可能是插入新 key，也可能是更新老 key），对相应位置进行赋值。

源码大体和之前讲的类似，核心还是一个双层循环，外层遍历 bucket 和它的 overflow bucket，内层遍历整个 bucket 的各个 cell

函数首先会检查 map 的标志位 flags。如果 flags 的写标志位此时被置 1 了，说明有其他协程在执行“写”操作，进而导致程序 panic。这也说明了 map 对协程是不安全的。

通过前文我们知道扩容是渐进式的，如果 map 处在扩容的过程中，那么当 key 定位到了某个 bucket 后，需要确保这个 bucket 对应的老 bucket 完成了迁移过程。即老 bucket 里的 key 都要迁移到新的 bucket 中来（分裂到 2 个新 bucket），才能在新的 bucket 中进行插入或者更新的操作。

上面说的操作是在函数靠前的位置进行的，只有进行完了这个搬迁操作后，我们才能放心地在新 bucket 里定位 key 要安置的地址，再进行之后的操作。

现在到了定位 key 应该放置的位置了，所谓找准自己的位置很重要。准备两个指针，一个（ inserti）指向 key 的 hash 值在 tophash 数组所处的位置，另一个( insertk)指向 cell 的位置（也就是 key 最终放置的地址），当然，对应 value 的位置就很容易定位出来了。这三者实际上都是关联的，在 tophash 数组中的索引位置决定了 key 在整个 bucket 中的位置（共 8 个 key），而 value 的位置需要“跨过” 8 个 key 的长度。

在循环的过程中，inserti 和 insertk 分别指向第一个找到的空闲的 cell。如果之后在 map 没有找到 key 的存在，也就是说原来 map 中没有此 key，这意味着插入新 key。那最终 key 的安置地址就是第一次发现的“空位”（tophash 是 empty）。

如果这个 bucket 的 8 个 key 都已经放置满了，那在跳出循环后，发现 inserti 和 insertk 都是空，这时候需要在 bucket 后面挂上 overflow bucket。当然，也有可能是在 overflow bucket 后面再挂上一个 overflow bucket。这就说明，太多 key hash 到了此 bucket。

在正式安置 key 之前，还要检查 map 的状态，看它是否需要进行扩容。如果满足扩容的条件，就主动触发一次扩容操作。

这之后，整个之前的查找定位 key 的过程，还得再重新走一次。因为扩容之后，key 的分布都发生了变化。

最后，会更新 map 相关的值，如果是插入新 key，map 的元素数量字段 count 值会加 1；在函数之初设置的 hashWriting 写标志出会清零。

### map的删除

写操作底层的执行函数是 mapdelete

```go
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) 
```



它首先会检查 h.flags 标志，如果发现写标位是 1，直接 panic，因为这表明有其他协程同时在进行写操作。

计算 key 的哈希，找到落入的 bucket。检查此 map 如果正在扩容的过程中，直接触发一次搬迁操作。

删除操作同样是两层循环，核心还是找到 key 的具体位置。寻找过程都是类似的，在 bucket 中挨个 cell 寻找。

找到对应位置后，对 key 或者 value 进行“清零”操作：
最后，将 count 值减 1，将对应位置的 tophash 值置成 Empty。

```go
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
  if raceenabled && h != nil {
    callerpc := getcallerpc()
    pc := funcPC(mapdelete)
    racewritepc(unsafe.Pointer(h), callerpc, pc)
    raceReadObjectPC(t.key, key, callerpc, pc)
  }
  if msanenabled && h != nil {
    msanread(key, t.key.size)
  }
  if h == nil || h.count == 0 {
    return
  }
  if h.flags&hashWriting != 0 {
    throw("concurrent map writes")
  }

  alg := t.key.alg
  hash := alg.hash(key, uintptr(h.hash0))

  // Set hashWriting after calling alg.hash, since alg.hash may panic,
  // in which case we have not actually done a write (delete).
  h.flags |= hashWriting

  bucket := hash & bucketMask(h.B)
  if h.growing() {
    growWork(t, h, bucket)
  }
  b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
  top := tophash(hash)
search:
  for ; b != nil; b = b.overflow(t) {
    for i := uintptr(0); i < bucketCnt; i++ {
      if b.tophash[i] != top {
        continue
      }
      k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
      k2 := k
      if t.indirectkey {
        k2 = *((*unsafe.Pointer)(k2))
      }
      if !alg.equal(key, k2) {
        continue
      }
      // Only clear key if there are pointers in it.
            // 对key清零
      if t.indirectkey {
        *(*unsafe.Pointer)(k) = nil
      } else if t.key.kind&kindNoPointers == 0 {
        memclrHasPointers(k, t.key.size)
      }
      v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
            // 对value清零
      if t.indirectvalue {
        *(*unsafe.Pointer)(v) = nil
      } else if t.elem.kind&kindNoPointers == 0 {
        memclrHasPointers(v, t.elem.size)
      } else {
        memclrNoHeapPointers(v, t.elem.size)
      }
            // 高位hash清零
      b.tophash[i] = empty
            // 个数减一
      h.count--
      break search
    }
  }

  if h.flags&hashWriting == 0 {
    throw("concurrent map writes")
  }
  h.flags &^= hashWriting
} 
```

Go 语言中只要是可比较的类型都可以作为 key。除开 slice，map，functions 这几种类型，其他类型都是 OK 的。具体包括：布尔值、数字、字符串、指针、通道、接口类型、结构体、只包含上述类型的数组。这些类型的共同特征是支持 == 和 != 操作符， k1==k2 时，可认为 k1 和 k2 是同一个 key。如果是结构体，则需要它们的字段值都相等，才被认为是相同的 key。
当用 float64 作为 key 的时候，先要将其转成 unit64 类型，再插入 key 中。
顺便说一句，任何类型都可以作为 value，包括 map 类型。

#### map的并发访问

**map 并不是一个线程安全的数据结构。同时读写一个 map 是不安全的，如果被检测到，会直接 panic。**

解决方法1：读写锁 sync.RWMutex。将map与读写锁定义在一个结构体，访问时加锁解锁

解决方法2：使用golang提供的 sync.Map

```go
func main() {
  m := sync.Map{}
  m.Store(1, 1)
  i := 0
  go func() {
    for i < 1000 {
      m.Store(1, 1)
      i++
    }
  }()

  go func() {
    for i < 1000 {
      m.Store(2, 2)
      i++
    }
  }()

  go func() {
    for i < 1000 {
      fmt.Println(m.Load(1))
      i++
    }
  }()

  for {
    runtime.GC()
  }
} 
```



最后看一看下列代码如果觉得和想的不一样，可以试试并想想为什么。

```go
 	var m map[string]string //m == nil
    delete(m,"name") //不会panic
    fmt.Println(m["name"]) //返回类型默认值
    m["name"] = "Li" //panic 
```



## 总结

总结一下，**Go 语言中，通过哈希查找表实现 map，用链表法解决哈希冲突**。

通过 key 的哈希值将 key 散落到不同的桶中，每个桶中有 8 个 cell。哈希值的低位决定桶序号，高位标识同一个桶中的不同 key。

当向桶中添加了很多 key，造成元素过多，或者溢出桶太多，就会触发扩容。扩容分为等量扩容和 2 倍容量扩容。扩容后，原来一个 bucket 中的 key 一分为二，会被重新分配到两个桶中。

扩容过程是渐进的，主要是防止一次扩容需要搬迁的 key 数量过多，引发性能问题。触发扩容的时机是增加了新元素，bucket 搬迁的时机则发生在赋值、删除期间，每次最多搬迁两个 bucket。

查找、赋值、删除的一个很核心的内容是如何定位到 key 所在的位置.