# RoaringBitmap Go 实现核心容器体系深度分析

> 本文档基于 `github.com/RoaringBitmap/roaring/v2` Go 实现源码进行深度分析，所有结论均给出对应文件与函数名，便于审阅者直接核对。

---

## 一、三种底层容器的存储结构与适用基数区间

### 1.1 存储结构总览

RoaringBitmap 将 32 位整数空间按高 16 位分片，每个分片（container）覆盖 65536（2^16）个值的范围。每个分片根据基数（cardinality）选择以下三种容器之一存储：

| 容器类型 | 结构体定义位置 | 核心字段 | 内存占用（近似） |
|---------|--------------|---------|----------------|
| `arrayContainer` | [arraycontainer.go:8-10](file:///e:/gsb/617/gsb_2/Jupiter/arraycontainer.go#L8-L10) | `content []uint16` 有序去重数组 | `2 * cardinality` 字节 |
| `bitmapContainer` | [bitmapcontainer.go:10-13](file:///e:/gsb/617/gsb_2/Jupiter/bitmapcontainer.go#L10-L13) | `cardinality int` + `bitmap []uint64`（1024 个 uint64） | 固定 8192 字节（8 * 1024） |
| `runContainer16` | [runcontainer.go:49-52](file:///e:/gsb/617/gsb_2/Jupiter/runcontainer.go#L49-L52) | `iv []interval16`（每个 interval 含 start + length） | `2 + 4 * n_runs` 字节 |

其中 `interval16` 定义于 [runcontainer.go:57-60](file:///e:/gsb/617/gsb_2/Jupiter/runcontainer.go#L57-L60)：
```go
type interval16 struct {
    start  uint16
    length uint16 // length minus 1
}
```
即每个 run 区间占用 4 字节（2 字节起始值 + 2 字节长度减一）。

### 1.2 转换阈值常量 `arrayDefaultMaxSize`

关键常量定义于 [util.go:10-17](file:///e:/gsb/617/gsb_2/Jupiter/util.go#L10-L17)：

```go
const (
    arrayDefaultMaxSize        = 4096 // containers with 4096 or fewer integers should be array containers.
    arrayLazyLowerBound        = 1024
    maxCapacity                = 1 << 16  // = 65536
    // ...
)
```

**阈值含义与比例关系：**

- `arrayDefaultMaxSize = 4096`：array 容器与 bitmap 容器的分界点
- `maxCapacity = 65536`：单个容器覆盖的全域基数（2^16）
- 比例：4096 / 65536 = **6.25%** 的密度阈值
  - 当基数 ≤ 4096（密度 ≤ 6.25%）：使用 arrayContainer，内存 ≤ 8KB，且随基数线性增长
  - 当基数 > 4096（密度 > 6.25%）：使用 bitmapContainer，固定 8KB 开销

bitmap 容器固定分配 1024 个 uint64 来覆盖全部 65536 位，见 [bitmapcontainer.go:23-28](file:///e:/gsb/617/gsb_2/Jupiter/bitmapcontainer.go#L23-L28) 的 `newBitmapContainer()`：
```go
func newBitmapContainer() *bitmapContainer {
    p := new(bitmapContainer)
    size := (1 << 16) / 64  // = 1024 words
    p.bitmap = make([]uint64, size, size)
    return p
}
```

### 1.3 array → bitmap 转换的精确触发条件

**1) iaddReturnMinimized 路径**（单个元素插入）：
见 [arraycontainer.go:290-314](file:///e:/gsb/617/gsb_2/Jupiter/arraycontainer.go#L290-L314)，当插入新元素后 `len(ac.content) >= arrayDefaultMaxSize` 时触发转换。

**2) iaddRange 路径**（范围插入）：
见 [arraycontainer.go:106-141](file:///e:/gsb/617/gsb_2/Jupiter/arraycontainer.go#L106-L141)，当 `newcardinality > arrayDefaultMaxSize` 时转换。

**3) iorArray 路径**（两个 array 容器原地 OR）：
见 [arraycontainer.go:388-419](file:///e:/gsb/617/gsb_2/Jupiter/arraycontainer.go#L388-L419)。

### 1.4 `iorArray` 中「只有确实超过阈值才转换」注释的深意

注释原文位于 [arraycontainer.go:412-417](file:///e:/gsb/617/gsb_2/Jupiter/arraycontainer.go#L412-L417)：

```go
if nl > arrayDefaultMaxSize {
    // Only converting to a bitmap when arrayDefaultMaxSize
    // is actually exceeded minimizes conversions in the case of repeated
    // calls to iorArray().
    return ac.toBitmapContainer()
}
return ac
```

**规避的性能问题：**

考虑一个边界场景：array1 有 4000 个元素，array2 有 300 个元素。它们的并集基数是 4050（假设部分重叠），仍 ≤ 4096，因此保持为 array。但如果我们「预判式」地只要 `maxPossibleCardinality > arrayDefaultMaxSize` 就先转 bitmap，再转回 array，就会发生 **bitmap → array 反向转换的额外开销**。

在重复调用 `iorArray()` 的场景中（例如批量聚合），如果每次都先升后降，会产生大量无意义的双向转换。当前实现坚持「实际基数真的超过才转换」，避免了「先膨胀为 bitmap 再收缩回 array」的振荡（thrashing）。

### 1.5 「DO NOT DO THIS」注释的含义

紧随其后的 `iorBitmap` 方法中（[arraycontainer.go:421-429](file:///e:/gsb/617/gsb_2/Jupiter/arraycontainer.go#L421-L429)）：

```go
func (ac *arrayContainer) iorBitmap(bc2 *bitmapContainer) container {
    bc1 := ac.toBitmapContainer()
    bc1.iorBitmap(bc2)
    // DO NOT DO THIS:
    // *ac = *newArrayContainerFromBitmap(bc1)
    // This will create gigantic array containers in the case of repeated calls to iorBitmap.
    return bc1
}
```

**规避的正确性/性能问题：**

如果 `iorBitmap` 把结果再转回 arrayContainer 并写回 `*ac`，在「重复调用」场景下会造成灾难性后果：

假设一个 bitmap 容器基数为 30000，与一个 array 容器做 OR。如果结果转回 array，就会产生一个包含 3 万个元素的 **超大 array 容器**——这严重违反了 array 容器只应在 ≤4096 基数时使用的设计原则。后续对这个超大 array 的任何操作（二分查找、插入、序列化）都会产生远高于 bitmap 容器的性能代价。

因此 `iorBitmap` 始终返回 bitmapContainer，绝不主动转回 array。反向转换（bitmap → array）只在 `iremove`、`iand` 等「基数肯定下降」的操作中，且确认基数回落到 ≤ 4096 时才会发生（例如 [bitmapcontainer.go:385-392](file:///e:/gsb/617/gsb_2/Jupiter/bitmapcontainer.go#L385-L392) 的 `iremoveReturnMinimized`）。

### 1.6 run 容器的适用场景

run 容器用于「连续值密集」的场景。`toEfficientContainer()` 方法会比较三种表示的序列化大小，选择最小者，见 [arraycontainer.go:1282-1295](file:///e:/gsb/617/gsb_2/Jupiter/arraycontainer.go#L1282-L1295) 和 [bitmapcontainer.go:1275-1290](file:///e:/gsb/617/gsb_2/Jupiter/bitmapcontainer.go#L1275-L1290)。

当单个容器表示全 1（65536 个连续值）时，会直接优化为 `runContainer16`（仅一个 run，仅 6 字节开销），见 [bitmapcontainer.go:366-372](file:///e:/gsb/617/gsb_2/Jupiter/bitmapcontainer.go#L366-L372) 的 `iaddReturnMinimized`。

---

## 二、写时复制（Copy-on-Write）机制

### 2.1 数据结构

`roaringArray` 是 Bitmap 的核心存储层，定义于 [roaringarray.go:121-126](file:///e:/gsb/617/gsb_2/Jupiter/roaringarray.go#L121-L126)：

```go
type roaringArray struct {
    keys            []uint16
    containers      []container `msg:"-"`
    needCopyOnWrite []bool
    copyOnWrite     bool
}
```

- `copyOnWrite`：全局标志，表示该 roaringArray 是否启用 COW 模式
- `needCopyOnWrite []bool`：**每个容器单独**的 COW 标志，与 `containers` 一一对应

### 2.2 COW 的置位时机

**1) Clone 时启用全局 COW**

见 [roaringarray.go:258-288](file:///e:/gsb/617/gsb_2/Jupiter/roaringarray.go#L258-L288) 的 `clone()` 方法：

```go
func (ra *roaringArray) clone() *roaringArray {
    sa := roaringArray{}
    sa.copyOnWrite = ra.copyOnWrite

    if ra.copyOnWrite {
        // 浅拷贝：共享底层容器
        sa.keys = make([]uint16, len(ra.keys))
        copy(sa.keys, ra.keys)
        sa.containers = make([]container, len(ra.containers))
        copy(sa.containers, ra.containers)  // 只拷贝指针，不拷贝容器数据
        sa.needCopyOnWrite = make([]bool, len(ra.needCopyOnWrite))
        ra.markAllAsNeedingCopyOnWrite()    // 源端全部标记为需要 COW
        sa.markAllAsNeedingCopyOnWrite()    // 目标端全部标记为需要 COW
    } else {
        // 深拷贝：克隆所有容器
        // ...
    }
    return &sa
}
```

关键：当 `copyOnWrite = true` 时，clone 只复制容器指针（`[]container` 切片的 copy），然后将源和目标的 **所有** `needCopyOnWrite[i]` 标记为 `true`。这样两个 roaringArray 就共享了底层容器数据。

**2) appendCopy 时传播 COW 标志**

见 [roaringarray.go:156-168](file:///e:/gsb/617/gsb_2/Jupiter/roaringarray.go#L156-L168)：

```go
func (ra *roaringArray) appendCopy(sa roaringArray, startingindex int) {
    copyonwrite := (ra.copyOnWrite && sa.copyOnWrite) || sa.needsCopyOnWrite(startingindex)
    if !copyonwrite {
        ra.appendContainer(sa.keys[startingindex], sa.containers[startingindex].clone(), copyonwrite)
    } else {
        ra.appendContainer(sa.keys[startingindex], sa.containers[startingindex], copyonwrite)
        if !sa.needsCopyOnWrite(startingindex) {
            sa.setNeedsCopyOnWrite(startingindex)  // 源端也标记
        }
    }
}
```

如果源容器已被标记为 `needCopyOnWrite`，或双方都启用了全局 COW，则新追加的容器共享数据并标记 COW。

**3) FromBuffer / FrozenView 时启用**

`FromBuffer` 和 `FrozenView` 等零拷贝反序列化方法会将所有容器标记为 `needCopyOnWrite = true`，因为容器数据直接引用输入字节切片，不能修改。见 [roaringarray.go:592](file:///e:/gsb/617/gsb_2/Jupiter/roaringarray.go#L592) 和 [serialization_littleendian.go:348](file:///e:/gsb/617/gsb_2/Jupiter/serialization_littleendian.go#L348)。

### 2.3 COW 的失效时机（清零）

`needCopyOnWrite[i]` 在 `getWritableContainerAtIndex` 中被清除，见 [roaringarray.go:348-354](file:///e:/gsb/617/gsb_2/Jupiter/roaringarray.go#L348-L354)：

```go
func (ra *roaringArray) getWritableContainerAtIndex(i int) container {
    if ra.needCopyOnWrite[i] {
        ra.containers[i] = ra.containers[i].clone()  // 真正发生拷贝
        ra.needCopyOnWrite[i] = false                // 此后不再需要 COW
    }
    return ra.containers[i]
}
```

即：**第一次写入时才真正克隆容器**，克隆后该位置的 COW 标志清零，后续写入直接修改本地副本。

### 2.4 `getWritableContainerAtIndex` vs `getContainerAtIndex`

| 方法 | 语义 | 适用场景 |
|-----|------|---------|
| `getContainerAtIndex(i)` | 返回原始容器指针，**只读**用途 | 仅读取容器内容时（遍历、基数查询、非原地运算） |
| `getWritableContainerAtIndex(i)` | 若 COW 标志为 true 则先克隆，返回可写副本 | 任何**原地修改**操作前必须调用 |

`getContainerAtIndex` 定义于 [roaringarray.go:317-319](file:///e:/gsb/617/gsb_2/Jupiter/roaringarray.go#L317-L319)，它直接返回 `ra.containers[i]`，不做任何检查。

### 2.5 不走 writable 路径直接改写的别名损坏后果

如果开发者绕过 `getWritableContainerAtIndex`，直接通过 `getContainerAtIndex` 获取指针并修改容器内容，会发生 **别名（aliasing）数据损坏**：

1. 两个 Bitmap（A 和 B）通过 Clone 共享同一个底层 arrayContainer C
2. A 直接获取 C 的指针并调用 `iadd(x)` 修改
3. B 毫不知情地继续使用 C，发现数据已被篡改
4. 更隐蔽的情况：A 和 B 可能在不同 goroutine 中并发修改 C，导致 data race

一个具体的损坏模式：A 在 C 的 content 切片中插入元素，触发了 Go 切片的扩容（append 超过 capacity），此时 A 的 content 指向新数组，但 B 仍然引用旧数组——A 的修改对 B 不可见。反之如果未扩容，则 B 会观察到 A 的修改。两种不一致行为都违反了值语义。

`cloneCopyOnWriteContainers()` 方法（[roaringarray.go:293-300](file:///e:/gsb/617/gsb_2/Jupiter/roaringarray.go#L293-L300)）用于强制克隆所有带 COW 标志的容器，常在释放底层缓冲区前调用。

---

## 三、惰性聚合路径（lazyOR / lazyIOR）

### 3.1 设计动机：为什么 lazy 更快？

批量 OR 聚合（如 `FastOr`）中，如果每一步都精确维护 `cardinality` 字段，会产生大量不必要的 popcount 计算。尤其对 bitmap 容器，OR 操作本身只是简单的按位或（O(1024) 次 word 运算），但为了更新基数需要额外执行一次 `popcntSlice`（同样 O(1024)）——等于开销翻倍。

惰性策略：**只做位运算，不更新基数**，全部聚合完成后再统一修复。

### 3.2 核心状态：`invalidCardinality`

常量定义于 [util.go:15](file:///e:/gsb/617/gsb_2/Jupiter/util.go#L15)：
```go
invalidCardinality = -1
```

在 lazy 路径中，bitmap 容器的 `cardinality` 字段被设为 `-1`，表示「基数无效，不可信任」。

例如 `lazyIORBitmap`（[bitmapcontainer.go:680-687](file:///e:/gsb/617/gsb_2/Jupiter/bitmapcontainer.go#L680-L687)）：
```go
func (bc *bitmapContainer) lazyIORBitmap(value2 *bitmapContainer) container {
    answer := bc
    for k := 0; k < len(answer.bitmap); k++ {
        answer.bitmap[k] = bc.bitmap[k] | value2.bitmap[k]
    }
    bc.cardinality = invalidCardinality  // 标记为无效
    return answer
}
```

对比非 lazy 的 `iorBitmap`（[bitmapcontainer.go:630-641](file:///e:/gsb/617/gsb_2/Jupiter/bitmapcontainer.go#L630-L641)），后者执行完 OR 后立即调用 `computeCardinality()` 更新基数。

### 3.3 `repairAfterLazy` 的作用

定义于 [fastaggregation.go:110-126](file:///e:/gsb/617/gsb_2/Jupiter/fastaggregation.go#L110-L126)：

```go
func (x1 *Bitmap) repairAfterLazy() {
    for pos := 0; pos < x1.highlowcontainer.size(); pos++ {
        c := x1.highlowcontainer.getContainerAtIndex(pos)
        switch c.(type) {
        case *bitmapContainer:
            if c.(*bitmapContainer).cardinality == invalidCardinality {
                c = x1.highlowcontainer.getWritableContainerAtIndex(pos)
                c.(*bitmapContainer).computeCardinality()  // 重新计算基数
                if c.(*bitmapContainer).getCardinality() <= arrayDefaultMaxSize {
                    x1.highlowcontainer.setContainerAtIndex(pos, c.(*bitmapContainer).toArrayContainer())
                } else if c.(*bitmapContainer).isFull() {
                    x1.highlowcontainer.setContainerAtIndex(pos, newRunContainer16Range(0, MaxUint16))
                }
            }
        }
    }
}
```

repair 做了三件事：
1. **计算基数**：对所有 `cardinality == -1` 的 bitmap 容器调用 `computeCardinality()`（基于 `popcntSlice`）
2. **向下转换**：如果基数回落到 ≤ 4096，转换为 array 容器
3. **向上优化**：如果是全满（65536），转换为 run 容器

### 3.4 `FastOr` 中的调用时机

见 [fastaggregation.go:150-163](file:///e:/gsb/617/gsb_2/Jupiter/fastaggregation.go#L150-L163)：

```go
func FastOr(bitmaps ...*Bitmap) *Bitmap {
    // ...
    answer := lazyOR(bitmaps[0], bitmaps[1])
    for _, bm := range bitmaps[2:] {
        answer = answer.lazyOR(bm)  // 全程 lazy
    }
    answer.repairAfterLazy()  // 最后统一 repair
    return answer
}
```

### 3.5 忘记 repair 的后果

如果调用方绕过 `FastOr` 直接使用 `lazyOR` / `lazyIOR` 且忘记调用 `repairAfterLazy`，会导致：

1. **基数查询完全错误**：`GetCardinality()` 返回 -1（对于 bitmap 容器，直接返回 `bc.cardinality` 字段，即 -1）
2. **依赖基数的操作全部异常**：例如 rank 运算、select 运算、容器转换判断等
3. **序列化失败**：`bitmapContainer.writeTo` 会检查 `cardinality <= arrayDefaultMaxSize` 并返回错误（[bitmapcontainer.go:20-25](file:///e:/gsb/617/gsb_2/Jupiter/bitmapcontainer.go#L20-L25) 注释提及，实际见 `writeTo` 逻辑）
4. **验证失败**：`Validate()` 会检查 cardinality 与实际 popcount 是否一致（[bitmapcontainer.go:1529-1531](file:///e:/gsb/617/gsb_2/Jupiter/bitmapcontainer.go#L1529-L1531)）

注意：在并行聚合 `ParOr` / `ParHeapOr` 中，每个 worker 内部都会调用 `repairAfterLazy`（[parallel.go:217](file:///e:/gsb/617/gsb_2/Jupiter/parallel.go#L217) 和 [parallel.go:399](file:///e:/gsb/617/gsb_2/Jupiter/parallel.go#L399)），所以最终结果已经被修复。

### 3.6 array 容器的 lazy 路径

array 容器的 lazy 方法目前直接转发到非 lazy 实现（例如 [arraycontainer.go:463-466](file:///e:/gsb/617/gsb_2/Jupiter/arraycontainer.go#L463-L466) 的 `lazyIorArray` 直接调用 `iorArray`）。惰性优化主要针对 bitmap 容器，因为 array 的基数就是 `len(content)`，维护成本为零。

但 `lazyorArray` 中有一个特殊点：使用 `arrayLazyLowerBound = 1024` 作为转 bitmap 的阈值（[arraycontainer.go:529](file:///e:/gsb/617/gsb_2/Jupiter/arraycontainer.go#L529)），比正常的 4096 更低。这是为了在 lazy 阶段更早地转为 bitmap，避免多次 array → array OR 的 O(n) 开销。

---

## 四、二进制序列化格式

### 4.1 Cookie 与格式版本

序列化格式遵循 [RoaringFormatSpec](https://github.com/RoaringBitmap/RoaringFormatSpec)，有两种 cookie 标识：

定义于 [util.go:14-16](file:///e:/gsb/617/gsb_2/Jupiter/util.go#L14-L16)：
```go
serialCookieNoRunContainer = 12346 // only arrays and bitmaps
serialCookie               = 12347 // runs, arrays, and bitmaps
```

- **12346**（无 run 容器格式）：4 字节 cookie + 4 字节容器数量
- **12347**（含 run 容器格式）：低 16 位为 cookie 值 12347，高 16 位为 `容器数量 - 1`（这样总共只用 4 字节）

序列化入口为 `roaringArray.writeTo`（[roaringarray.go:493-567](file:///e:/gsb/617/gsb_2/Jupiter/roaringarray.go#L493-L567)），反序列化为 `roaringArray.readFrom`（[roaringarray.go:577-703](file:///e:/gsb/617/gsb_2/Jupiter/roaringarray.go#L577-L703)）。

### 4.2 含 run 容器时的 isRun 位图

当存在 run 容器时（`cookie == 12347`），cookie 之后紧跟一个 **isRun 位图**，每一位标记对应位置的容器是否为 run 类型。位图大小为 `(size + 7) / 8` 字节。

见序列化端 [roaringarray.go:508-521](file:///e:/gsb/617/gsb_2/Jupiter/roaringarray.go#L508-L521)：
```go
if hasRun {
    binary.LittleEndian.PutUint16(buf[0:], uint16(serialCookie))
    binary.LittleEndian.PutUint16(buf[2:], uint16(len(ra.keys)-1))
    // ...
    runbitmapslice := buf[nw : nw+isRunSizeInBytes]
    for i, c := range ra.containers {
        switch c.(type) {
        case *runContainer16:
            runbitmapslice[i/8] |= 1 << (uint(i) % 8)
        }
    }
}
```

反序列化端 [roaringarray.go:597-604](file:///e:/gsb/617/gsb_2/Jupiter/roaringarray.go#L597-L604) 读取该位图后，根据每一位决定如何解析后续容器数据。

### 4.3 描述头与基数信息

isRun 位图之后是 **描述头（descriptive header）**，每个容器占 4 字节：
- 2 字节：key（高 16 位）
- 2 字节：`cardinality - 1`

见 [roaringarray.go:530-536](file:///e:/gsb/617/gsb_2/Jupiter/roaringarray.go#L530-L536)。

用 `cardinality - 1` 而非 `cardinality` 是因为：基数最小为 1，减 1 后可以用 uint16 完整表示 1~65536 的范围（0 表示 1，65535 表示 65536）。

### 4.4 偏移量目录与 `noOffsetThreshold`

描述头之后是可选的 **偏移量目录（offset header）**，保存每个容器数据在文件中的起始位置。但当容器数量 < 4 时，偏移量目录被省略以节省空间。

常量定义于 [util.go:17](file:///e:/gsb/617/gsb_2/Jupiter/util.go#L17)：
```go
noOffsetThreshold = 4
```

见序列化端 [roaringarray.go:538-551](file:///e:/gsb/617/gsb_2/Jupiter/roaringarray.go#L538-L551)：
```go
if !hasRun || (len(ra.keys) >= noOffsetThreshold) {
    // offset header
    for _, c := range ra.containers {
        binary.LittleEndian.PutUint32(buf[nw:], uint32(startOffset))
        // ...
    }
}
```

反序列化端如果有偏移量目录就直接 `SkipBytes` 跳过（[roaringarray.go:626-630](file:///e:/gsb/617/gsb_2/Jupiter/roaringarray.go#L626-L630)），因为按顺序读取不需要随机访问。

### 4.5 大小端处理

**序列化格式固定为小端序（little-endian）**，所有多字节字段（cookie、key、cardinality、offset、bitmap 数据等）均使用小端。

证据：
- 序列化使用 `binary.LittleEndian.PutUint16/PutUint32`（见 `writeTo` 各处）
- 反序列化使用 `binary.LittleEndian.Uint32` 读取 cookie（见 `readFrom`）
- `bitmapContainer` 的数据直接以 `uint64` 切片的原始内存布局写入（[serialization_littleendian.go:19-25](file:///e:/gsb/617/gsb_2/Jupiter/serialization_littleendian.go#L19-L25)），而该文件有 build tag 限制在小端架构上编译：
  ```go
  //go:build (386 && !appengine) || (amd64 && !appengine) || ...
  ```

`serialization_generic.go` 提供大端架构的回退实现（通过 `encoding/binary` 逐字转换）。因此跨架构读写是兼容的——格式总是小端，大端机器上用通用路径做字节序转换。

### 4.6 运行时无 run 容器的优化

如果 bitmap 不含任何 run 容器，序列化时会选择 `serialCookieNoRunContainer`（值 12346）格式。这省去了 isRun 位图的空间（对于大量容器的场景很可观），并且使用独立的 4 字节存储 size，而不是把 size 打包进 cookie 的高 16 位。

判断逻辑见 `hasRunCompression()`（[roaringarray.go:705-713](file:///e:/gsb/617/gsb_2/Jupiter/roaringarray.go#L705-L713)），遍历所有容器检查是否存在 `*runContainer16` 类型。

---

## 五、容易被忽视的潜在风险与陷阱

### 风险一：非并发安全——任何共享都需要外部同步

**代码依据：** 整个代码库中没有任何 mutex、atomic 或其他同步原语。`Bitmap` 结构体（[roaring.go:20-22](file:///e:/gsb/617/gsb_2/Jupiter/roaring.go#L20-L22)）及其内部的 `roaringArray`、各容器结构体都没有锁。

具体危险场景：
- 一个 goroutine 读（`Contains`、`GetCardinality`），另一个 goroutine 写（`Add`、`Remove`）→ data race
- 两个 goroutine 同时写同一个 bitmap → 内部切片扩容、容器替换时的竞态
- 两个 goroutine 同时对 COW 共享的容器做「首次写入」→ 可能产生两份独立的拷贝，逻辑上不影响正确性，但浪费内存（不过需要确认，因为 Go 的 map 不是并发安全的，但这里的 slice 写是有问题的）

更隐蔽的是 **Clone 后的并发修改**：如果 A 和 B 两个 goroutine 各持有 Clone 得到的 bitmap，它们看似独立，但在 COW 生效期间，它们共享底层容器。如果两边同时触发对同一容器的首次写入（`getWritableContainerAtIndex`），对 `ra.containers[i]` 和 `ra.needCopyOnWrite[i]` 的写操作不是原子的——理论上可能两个 goroutine 都认为需要克隆，都执行克隆，但各自写入不同的副本，导致后续数据不一致。

### 风险二：COW 下的共享别名——`getContainerAtIndex` 的误用

**代码依据：** `getContainerAtIndex`（[roaringarray.go:317-319](file:///e:/gsb/617/gsb_2/Jupiter/roaringarray.go#L317-L319)）直接返回容器指针，不检查 COW 标志。而 `getWritableContainerAtIndex`（[roaringarray.go:348-354](file:///e:/gsb/617/gsb_2/Jupiter/roaringarray.go#L348-L354)）才是写入前应调用的。

对于库的使用者来说，虽然一般不会直接调用 `getContainerAtIndex`，但以下场景仍可能踩坑：

1. **`FromBuffer` / `FromUnsafeBytes` 返回的 bitmap 直接被修改底层数据**
   - 这些函数返回的 bitmap 所有容器都引用输入字节切片（零拷贝）
   - 如果调用者修改了原始 `buf`，bitmap 的数据会随之改变
   - 文档（[roaring.go:364-373](file:///e:/gsb/617/gsb_2/Jupiter/roaring.go#L364-L373)）明确警告，但很容易被忽视

2. **派生 bitmap 的传播性 COW**
   - A 由 `FromBuffer` 创建（全 COW）
   - B = Or(A, C)，其中部分容器来自 A 的拷贝（appendCopy 路径）
   - 如果此时释放 A 的底层 buffer，B 中那些仍共享 A 容器的部分会变成悬空引用
   - 文档建议调用 `CloneCopyOnWriteContainers()`（[roaringarray.go:293-300](file:///e:/gsb/617/gsb_2/Jupiter/roaringarray.go#L293-L300)）来强制脱离

### 风险三：`lazyOR` / `lazyIOR` 必须配合 `repairAfterLazy` 使用

**代码依据：** `lazyOR` 和 `lazyIOR` 是公开方法（`Bitmap` 级别），定义于 [fastaggregation.go:8-53](file:///e:/gsb/617/gsb_2/Jupiter/fastaggregation.go#L8-L53) 和 [fastaggregation.go:56-107](file:///e:/gsb/617/gsb_2/Jupiter/fastaggregation.go#L56-L107)。

虽然 `FastOr` 会自动调用 `repairAfterLazy`，但如果使用者直接调用 `lazyOR` 而忘记 repair，会得到一个「看起来正常但基数为 -1」的 bitmap。后续调用：
- `GetCardinality()` 返回无意义的负数或溢出
- `WriteTo()` 可能失败或写出错误数据
- `Validate()` 会报错

这是一个「部分有效」的状态——迭代遍历可能正常（因为遍历不依赖 cardinality 字段），但依赖基数的操作全部异常，难以调试。

### 风险四：并行聚合对输入 bitmap 的隐含假设

**代码依据：** `ParOr`（[parallel.go:337-454](file:///e:/gsb/617/gsb_2/Jupiter/parallel.go#L337-L454)）和 `ParHeapOr`、`ParAnd` 使用多个 goroutine 并发处理。

`ParOr` 的并行策略是按 key 范围分片。但它在 `lazyIOrOnRange` 中（[parallel.go:554-612](file:///e:/gsb/617/gsb_2/Jupiter/parallel.go#L554-L612)）会直接修改第一个参数的 `ra1`（原地 OR）。在并行上下文中，每个 chunk 由独立的 `roaringArray` 累积，然后合并到结果中——这部分是安全的。

但更值得注意的是 **输入 bitmap 的并发读取安全假设**：并行函数默认输入 bitmap 在聚合期间不会被修改。如果某个 goroutine 在 `ParOr` 运行时修改了输入 bitmap 的某个容器，而 worker goroutine 正在读取该容器，就会产生 data race。

此外，`ParOr` 中调用 `ra1.insertNewKeyValueAt`（[parallel.go:575](file:///e:/gsb/617/gsb_2/Jupiter/parallel.go#L575)）和对 `ra1.containers[idx1]` 的直接赋值，如果 ra1 是某个共享 bitmap 的一部分，也会产生别名问题。

### 风险五：序列化跨架构的字节序与结构体对齐陷阱

**代码依据：** `serialization_littleendian.go` 使用 `unsafe.Slice` 直接将 `[]uint64`  reinterpret 为 `[]byte`（[serialization_littleendian.go:27-34](file:///e:/gsb/617/gsb_2/Jupiter/serialization_littleendian.go#L27-L34)），仅在小端架构编译。

虽然 `serialization_generic.go` 提供了大端回退，保证了**格式层面**的跨架构兼容（文件总是小端），但有一个更深层的陷阱：

**`runContainer16` 的 `interval16` 结构体内存布局。** `interval16` 包含两个 `uint16` 字段（start 和 length），在 Go 中没有 padding，理论上是 4 字节连续。但 `interval16SliceAsByteSlice` 也是用 unsafe 直接转换（[serialization_littleendian.go:45-52](file:///e:/gsb/617/gsb_2/Jupiter/serialization_littleendian.go#L45-L52)），依赖于：
1. 结构体字段顺序与序列化顺序一致
2. 字段之间无 padding
3. 主机字节序为小端

其中第 2 点（无 padding）对于 `uint16` + `uint16` 是成立的，但如果未来有人修改 `interval16` 结构体（如添加字段或调整顺序）而未同步修改序列化代码，就会产生静默的二进制不兼容。

`FrozenView` / `Freeze` 格式（CRoaring frozen format）同样大量使用 unsafe 指针转换，对结构体布局有严格依赖。

---

## 六、关键常量速查表

| 常量 | 值 | 含义 | 定义位置 |
|-----|----|------|---------|
| `arrayDefaultMaxSize` | 4096 | array/bitmap 容器分界基数 | [util.go:11](file:///e:/gsb/617/gsb_2/Jupiter/util.go#L11) |
| `arrayLazyLowerBound` | 1024 | lazy 路径下 array→bitmap 更低阈值 | [util.go:12](file:///e:/gsb/617/gsb_2/Jupiter/util.go#L12) |
| `maxCapacity` | 65536 | 单容器全域基数 | [util.go:13](file:///e:/gsb/617/gsb_2/Jupiter/util.go#L13) |
| `invalidCardinality` | -1 | lazy 模式下的无效基数标记 | [util.go:15](file:///e:/gsb/617/gsb_2/Jupiter/util.go#L15) |
| `serialCookieNoRunContainer` | 12346 | 无 run 容器的序列化 cookie | [util.go:14](file:///e:/gsb/617/gsb_2/Jupiter/util.go#L14) |
| `serialCookie` | 12347 | 含 run 容器的序列化 cookie | [util.go:16](file:///e:/gsb/617/gsb_2/Jupiter/util.go#L16) |
| `noOffsetThreshold` | 4 | 省略偏移量目录的容器数阈值 | [util.go:17](file:///e:/gsb/617/gsb_2/Jupiter/util.go#L17) |
| `MaxNumIntervals` | 2048 | run 容器最大区间数 | [runcontainer.go:68](file:///e:/gsb/617/gsb_2/Jupiter/runcontainer.go#L68) |
