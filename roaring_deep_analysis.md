# RoaringBitmap (Go) 核心容器体系深度代码分析

---

## 一、三种底层容器的存储结构与适用基数区间

### 1.1 arrayContainer

**文件**: [arraycontainer.go](file:///e:/gsb/617/gsb_2/Mercury/arraycontainer.go)

```go
type arrayContainer struct {
    content []uint16
}
```

- **存储结构**: 有序、去重的 `[]uint16` 动态数组，每个元素直接存储一个被设置的 16 位整数值。
- **内存占用**: `2 × cardinality` 字节（见 `getSizeInBytes`，[arraycontainer.go:93](file:///e:/gsb/617/gsb_2/Mercury/arraycontainer.go#L93)）。
- **适用基数区间**: `[0, arrayDefaultMaxSize]`，即 `[0, 4096]`。当基数超过 4096 时应转换为 bitmapContainer。
- **getCardinality 实现**: 直接返回 `len(ac.content)`（[arraycontainer.go:950](file:///e:/gsb/617/gsb_2/Mercury/arraycontainer.go#L950)），O(1)。

### 1.2 bitmapContainer

**文件**: [bitmapcontainer.go](file:///e:/gsb/617/gsb_2/Mercury/bitmapcontainer.go)

```go
type bitmapContainer struct {
    cardinality int
    bitmap      []uint64
}
```

- **存储结构**: 固定 1024 个 `uint64`（即 `65536 / 64`）的位图，加上一个缓存的 `cardinality` 字段。
- **内存占用**: 固定 `1024 × 8 = 8192` 字节（见 `bitmapContainerSizeInBytes`，[bitmapcontainer.go:309](file:///e:/gsb/617/gsb_2/Mercury/bitmapcontainer.go#L309)），与基数无关。
- **适用基数区间**: `(arrayDefaultMaxSize, 65536]`，即 `(4096, 65536]`。当基数降至 `≤ 4096` 时应转换为 arrayContainer。
- **getCardinality 实现**: 直接返回缓存的 `bc.cardinality` 字段（[bitmapcontainer.go:408](file:///e:/gsb/617/gsb_2/Mercury/bitmapcontainer.go#L408)），O(1)；当缓存失效时需调用 `computeCardinality`（[bitmapcontainer.go:611](file:///e:/gsb/617/gsb_2/Mercury/bitmapcontainer.go#L611)），遍历 1024 个 uint64 做 popcnt，O(1024)。

### 1.3 runContainer16

**文件**: [runcontainer.go](file:///e:/gsb/617/gsb_2/Mercury/runcontainer.go)

```go
type runContainer16 struct {
    iv []interval16
}

type interval16 struct {
    start  uint16
    length uint16 // length minus 1
}
```

- **存储结构**: 有序、不重叠、不相邻的 `[]interval16`，每个 interval16 存储 `[start, start+length]` 的闭区间（`length` 字段存储的是实际长度减 1）。
- **内存占用**: `2 + 4 × numRuns` 字节（见 `serializedSizeInBytes`，[runcontainer.go:2787](file:///e:/gsb/617/gsb_2/Mercury/runcontainer.go#L2787)）。
- **适用基数区间**: 不由基数单独决定，而是由**游程（run）数量**与基数共同决定。当 `sizeAsRunContainer < min(sizeAsBitmapContainer, sizeAsArrayContainer)` 时使用 runContainer（见 `toEfficientContainer`，[runcontainer.go:2689](file:///e:/gsb/617/gsb_2/Mercury/runcontainer.go#L2689)）。具体地：
  - 与 bitmap 比较：`4 × numRuns + 2 < 8192` → `numRuns < 2047.5`，即 `numRuns ≤ 2047`。
  - 与 array 比较：`4 × numRuns + 2 < 2 × cardinality` → `numRuns < (cardinality - 1) / 2`。
- **getCardinality 实现**: 遍历所有 interval 累加 `runlen()`（[runcontainer.go:943](file:///e:/gsb/617/gsb_2/Mercury/runcontainer.go#L943)），O(numRuns)，非 O(1)。

### 1.4 array 与 bitmap 之间相互转换的精确触发条件与阈值常量

**关键常量**（[util.go:11](file:///e:/gsb/617/gsb_2/Mercury/util.go#L11)）：

```go
arrayDefaultMaxSize = 4096
maxCapacity          = 1 << 16  // 65536
```

**array → bitmap 转换触发条件**：

| 场景 | 代码位置 | 精确条件 |
|------|---------|---------|
| `iaddReturnMinimized` | [arraycontainer.go:301](file:///e:/gsb/617/gsb_2/Mercury/arraycontainer.go#L301) | `len(ac.content) >= arrayDefaultMaxSize` 且新元素不在已有数组中 |
| `iaddRange` | [arraycontainer.go:122](file:///e:/gsb/617/gsb_2/Mercury/arraycontainer.go#L122) | `newcardinality > arrayDefaultMaxSize` |
| `orArray` | [arraycontainer.go:496](file:///e:/gsb/617/gsb_2/Mercury/arraycontainer.go#L496) | `maxPossibleCardinality > arrayDefaultMaxSize` |
| `iorArray` | [arraycontainer.go:412](file:///e:/gsb/617/gsb_2/Mercury/arraycontainer.go#L412) | 合并后 `nl > arrayDefaultMaxSize` |
| `notClose` | [arraycontainer.go:193](file:///e:/gsb/617/gsb_2/Mercury/arraycontainer.go#L193) | `newCardinality > arrayDefaultMaxSize` |
| `xorArray` | [arraycontainer.go:658](file:///e:/gsb/617/gsb_2/Mercury/arraycontainer.go#L658) | `totalCardinality > arrayDefaultMaxSize` |

**bitmap → array 转换触发条件**：

| 场景 | 代码位置 | 精确条件 |
|------|---------|---------|
| `iremoveReturnMinimized` | [bitmapcontainer.go:387](file:///e:/gsb/617/gsb_2/Mercury/bitmapcontainer.go#L387) | `bc.cardinality == arrayDefaultMaxSize`（精确等于 4096） |
| `iremoveRange` | [bitmapcontainer.go:431](file:///e:/gsb/617/gsb_2/Mercury/bitmapcontainer.go#L431) | `bc.getCardinality() <= arrayDefaultMaxSize` |
| `inot` | [bitmapcontainer.go:448](file:///e:/gsb/617/gsb_2/Mercury/bitmapcontainer.go#L448) | `bc.getCardinality() <= arrayDefaultMaxSize` |
| `andBitmap` | [bitmapcontainer.go:871](file:///e:/gsb/617/gsb_2/Mercury/bitmapcontainer.go#L871) | `newcardinality <= arrayDefaultMaxSize` |
| `xorBitmap` | [bitmapcontainer.go:747](file:///e:/gsb/617/gsb_2/Mercury/bitmapcontainer.go#L747) | `newCardinality <= arrayDefaultMaxSize` |
| `toEfficientContainer` | [bitmapcontainer.go:1286](file:///e:/gsb/617/gsb_2/Mercury/bitmapcontainer.go#L1286) | `card <= arrayDefaultMaxSize` |

**`arrayDefaultMaxSize` 与 65536 全域基数的关系**：

- `arrayDefaultMaxSize = 4096 = 65536 / 16`，恰好是全域基数的 1/16。
- 这个值的选择基于存储效率交叉点：array 容器每元素 2 字节，当基数 ≤ 4096 时占用 `≤ 8192` 字节，不超过 bitmap 的固定 8192 字节；基数 > 4096 时 array 更大，应切换为 bitmap。
- 注意 `iremoveReturnMinimized` 使用的是**精确等于**（`==`）而非 `<=`，这是因为 `iremove` 每次只删除一个元素，cardinality 只能从 4097 降到 4096。

### 1.5 iorArray 中「只有在确实超过阈值时才转换为 bitmap」及「DO NOT DO THIS」注释分析

**iorArray 代码**（[arraycontainer.go:388-419](file:///e:/gsb/617/gsb_2/Mercury/arraycontainer.go#L388)）：

```go
func (ac *arrayContainer) iorArray(value2 *arrayContainer) container {
    // ... 合并逻辑 ...
    nl := union2by2(...)
    ac.content = ac.content[:nl]

    if nl > arrayDefaultMaxSize {
        // Only converting to a bitmap when arrayDefaultMaxSize
        // is actually exceeded minimizes conversions in the case of repeated
        // calls to iorArray().
        return ac.toBitmapContainer()
    }
    return ac
}
```

**注释含义**: 只有当合并后的实际基数 `nl` **确实超过** `arrayDefaultMaxSize` 时才转换为 bitmap。这是为了在**反复调用 `iorArray`** 的场景下最小化容器类型转换次数。如果使用更激进的策略（例如在 `maxPossibleCardinality > arrayDefaultMaxSize` 时就提前转换），那么在多次增量 OR 操作中，每次都可能触发不必要的 array→bitmap→array 往返转换。

**iorBitmap 中的「DO NOT DO THIS」注释**（[arraycontainer.go:422-429](file:///e:/gsb/617/gsb_2/Mercury/arraycontainer.go#L422)）：

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

**规避的问题**: 如果在 OR 操作后将 bitmap 转回 array（即注释掉的那行代码），在**反复调用 `iorBitmap`** 的场景下会产生灾难性的性能退化。原因如下：

1. **正确性/性能问题**: 假设 array 有 100 个元素，bitmap 有 50000 个元素。OR 后结果约 50000 个元素，是 bitmap 容器。如果将其转回 array，则下次再 `iorBitmap` 时，又需要将这个巨大的 array 转为 bitmap，然后再转回 array——每次调用都产生 O(50000) 的转换开销。
2. **别名问题**: `*ac = *newArrayContainerFromBitmap(bc1)` 是结构体值拷贝，会将 bitmapContainer 的字段覆盖到 arrayContainer 的字段上。由于 Go 中这两种结构体字段不同（`bitmapContainer` 有 `cardinality int` + `bitmap []uint64`，`arrayContainer` 只有 `content []uint16`），这种赋值会导致**内存布局错位**和**未定义行为**。更根本的是，方法签名返回 `container` 接口，调用方期望拿到的是正确的容器类型，而 `*ac = *...` 会把 `ac` 指针本身从 arrayContainer 变成 bitmapContainer 的内存布局，这在 Go 的类型系统中是**未定义行为**——`ac` 声明为 `*arrayContainer`，但其底层内存被覆盖为 `bitmapContainer` 的布局，会导致严重的内存损坏。

因此，正确的做法是**直接返回 bitmapContainer**，让调用方在更高层决定是否需要降级。

---

## 二、roaringarray 的写时复制（Copy-on-Write）实现

### 2.1 COW 核心数据结构

**文件**: [roaringarray.go](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go)

```go
type roaringArray struct {
    keys            []uint16
    containers      []container
    needCopyOnWrite []bool
    copyOnWrite     bool
}
```

- `copyOnWrite`: 位图级别的全局 COW 开关，由 `SetCopyOnWrite` 设置（[roaring.go:2170](file:///e:/gsb/617/gsb_2/Mercury/roaring.go#L2170)）。
- `needCopyOnWrite[]`: 逐容器的 COW 标志，表示该容器是否被多个 roaringArray 共享。

### 2.2 needCopyOnWrite 标志的置位时机

| 时机 | 代码位置 | 说明 |
|------|---------|------|
| `clone()` | [roaringarray.go:270-271](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L270) | 当 `copyOnWrite=true` 时，clone 不深拷贝容器，而是将源和副本的所有容器 `needCopyOnWrite` 全部置为 `true` |
| `appendCopy` | [roaringarray.go:164-166](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L164) | 当 COW 生效时，共享容器引用，并将源容器的 `needCopyOnWrite` 置为 `true` |
| `appendCopiesUntil/After` | [roaringarray.go:193-194](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L193), [roaringarray.go:219-220](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L219) | 同上，共享时置位 |
| `FromBuffer` / `FrozenView` | [roaringarray.go:655](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L655), [serialization_littleendian.go:349](file:///e:/gsb/617/gsb_2/Mercury/serialization_littleendian.go#L349) | 零拷贝反序列化时，所有容器 `needCopyOnWrite` 置为 `true`（因为底层 buffer 可能被共享） |

### 2.3 needCopyOnWrite 标志的失效（清除）时机

| 时机 | 代码位置 | 说明 |
|------|---------|------|
| `getWritableContainerAtIndex` | [roaringarray.go:349-353](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L349) | 当需要写入时，若 `needCopyOnWrite[i]=true`，则深拷贝容器并置 `needCopyOnWrite[i]=false` |
| `cloneCopyOnWriteContainers` | [roaringarray.go:294-299](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L294) | 批量将所有 `needCopyOnWrite=true` 的容器深拷贝并清除标志 |
| `insertNewKeyValueAt` | [roaringarray.go:382-384](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L382) | 新插入的容器 `needCopyOnWrite` 初始化为 `false` |

### 2.4 getWritableContainerAtIndex 与 getContainerAtIndex 的语义差别

**`getContainerAtIndex(i)`**（[roaringarray.go:317](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L317)）：
- 直接返回 `ra.containers[i]`，**不做任何 COW 检查或拷贝**。
- 语义：获取容器的只读引用。

**`getWritableContainerAtIndex(i)`**（[roaringarray.go:348-354](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L348)）：
```go
func (ra *roaringArray) getWritableContainerAtIndex(i int) container {
    if ra.needCopyOnWrite[i] {
        ra.containers[i] = ra.containers[i].clone()
        ra.needCopyOnWrite[i] = false
    }
    return ra.containers[i]
}
```
- 若该容器被标记为需要 COW，则**先深拷贝再返回**新容器的引用。
- 语义：获取一个**可安全写入**的容器引用，保证写入不会影响其他共享同一容器的 roaringArray。

### 2.5 COW 共享底层容器后直接改写的别名数据损坏

假设 Bitmap A 和 Bitmap B 通过 COW 共享了同一个容器 `c`（`A.containers[i]` 和 `B.containers[j]` 都指向同一个 `*arrayContainer`），且 `needCopyOnWrite` 都为 `true`。

如果对 A 执行写操作时**不走 `getWritableContainerAtIndex` 路径**，而是直接通过 `getContainerAtIndex` 获取引用后修改：

```go
c := A.highlowcontainer.getContainerAtIndex(i)  // 返回共享的 *arrayContainer
c.iadd(42)  // 直接修改了共享的底层 slice
```

那么 B 中对应的容器也会被修改，因为两者共享同一个 `*arrayContainer` 指针和其内部的 `content []uint16` slice。这会导致：

1. **B 的数据被静默损坏**：B 中出现了本不应存在的值 42。
2. **B 的基数不一致**：B 的 `needCopyOnWrite[j]` 仍为 `true`，但容器内容已被修改，后续 B 再通过 `getWritableContainerAtIndex` 获取时会 clone 已被污染的容器。
3. **并发安全幻觉**：使用者可能认为 A 和 B 是独立的位图，但实际上它们的内部状态已耦合。

这正是 `getWritableContainerAtIndex` 存在的原因——它确保在写入前打破共享。

---

## 三、惰性聚合路径（lazyOR / lazyIOR）

### 3.1 惰性聚合为什么能提升批量聚合性能

**核心思想**（来自 [runcontainer.go:2366-2407](file:///e:/gsb/617/gsb_2/Mercury/runcontainer.go#L2366) 中 @lemire 的注释）：

在批量聚合（例如 OR 100 个位图）的过程中，中间结果不需要处于"最优压缩"状态。只需要最终结果是最优的。因此：

1. **跳过基数计算**: 在 lazy 阶段，bitmapContainer 的 OR 操作只做位运算（`bitmap[i] |= other.bitmap[i]`），**不计算合并后的 cardinality**。正常 OR 每次合并后都要做 `computeCardinality`（遍历 1024 个 uint64 做 popcnt），这在批量聚合中是 O(N × 1024) 的冗余开销。
2. **跳过容器类型降级**: lazy 阶段不检查 bitmap 是否应降级为 array，避免了中间结果的反复类型转换。
3. **延迟到最终一次性修复**: 所有 OR 完成后，调用一次 `repairAfterLazy`，统一计算 cardinality 并做容器类型优化。

**性能对比**:
- 非 lazy：N 次 OR → N 次 `computeCardinality` + N 次类型检查/转换
- lazy：N 次 OR → 0 次 `computeCardinality` + 最后 1 次 `repairAfterLazy`

### 3.2 lazy 阶段 bitmapContainer 的 cardinality 字段状态

在 lazy 阶段，bitmapContainer 的 `cardinality` 字段被设置为 `invalidCardinality = -1`（[util.go:15](file:///e:/gsb/617/gsb_2/Mercury/util.go#L15)）。

**不可信**。以下代码明确标记了这一点：

- `lazyIORBitmap`（[bitmapcontainer.go:685](file:///e:/gsb/617/gsb_2/Mercury/bitmapcontainer.go#L685)）：`bc.cardinality = invalidCardinality`
- `lazyORArray`（[bitmapcontainer.go:671](file:///e:/gsb/617/gsb_2/Mercury/bitmapcontainer.go#L671)）：`answer.cardinality = invalidCardinality`
- `lazyIORArray`（[bitmapcontainer.go:671](file:///e:/gsb/617/gsb_2/Mercury/bitmapcontainer.go#L671)）：`answer.cardinality = invalidCardinality`
- `lazyIOR` 对 runContainer16（[bitmapcontainer.go:540](file:///e:/gsb/617/gsb_2/Mercury/bitmapcontainer.go#L540)）：`bc.cardinality = invalidCardinality`

### 3.3 repairAfterLazy / computeCardinality 的作用

**`computeCardinality`**（[bitmapcontainer.go:611-613](file:///e:/gsb/617/gsb_2/Mercury/bitmapcontainer.go#L611)）：
```go
func (bc *bitmapContainer) computeCardinality() {
    bc.cardinality = int(popcntSlice(bc.bitmap))
}
```
遍历 1024 个 uint64，通过 popcnt 恢复准确的 cardinality。

**`repairAfterLazy`**（[fastaggregation.go:110-126](file:///e:/gsb/617/gsb_2/Mercury/fastaggregation.go#L110)）：
```go
func (x1 *Bitmap) repairAfterLazy() {
    for pos := 0; pos < x1.highlowcontainer.size(); pos++ {
        c := x1.highlowcontainer.getContainerAtIndex(pos)
        switch c.(type) {
        case *bitmapContainer:
            if c.(*bitmapContainer).cardinality == invalidCardinality {
                c = x1.highlowcontainer.getWritableContainerAtIndex(pos)
                c.(*bitmapContainer).computeCardinality()
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

`repairAfterLazy` 做三件事：
1. 找出所有 `cardinality == invalidCardinality` 的 bitmapContainer。
2. 调用 `computeCardinality` 恢复准确基数。
3. 根据恢复后的基数做容器类型优化：若 `≤ 4096` 降级为 array；若 `== 65536` 升级为 full runContainer。

### 3.4 忘记执行 repair 会得到的错误结果

如果调用方使用 `lazyOR`/`lazyIOR` 后忘记调用 `repairAfterLazy`：

1. **`GetCardinality()` 返回错误值**: `bitmapContainer.getCardinality()` 直接返回 `cardinality` 字段，此时为 `-1`。上层 `Bitmap.GetCardinality()`（[roaring.go:1247](file:///e:/gsb/617/gsb_2/Mercury/roaring.go#L1247)）会累加所有容器的 cardinality，导致结果偏小甚至为负数。
2. **`IsEmpty()` 可能误判**: 若某容器实际有值但 cardinality 为 -1，`isEmpty()` 返回 `false`，但 `GetCardinality()` 可能返回异常值。
3. **`Rank()` / `Select()` 等查询结果错误**: 这些操作依赖准确的 cardinality。
4. **序列化可能失败**: `bitmapContainer.writeTo`（[serialization_littleendian.go:20](file:///e:/gsb/617/gsb_2/Mercury/serialization_littleendian.go#L20)）会检查 `bc.cardinality <= arrayDefaultMaxSize` 并拒绝写入，导致序列化报错 `"refusing to write bitmap container with cardinality of array container"`。
5. **容器类型不优**: bitmap 可能本应降级为 array 但未降级，浪费内存。

**注意**: 公共 API `FastOr`（[fastaggregation.go:150](file:///e:/gsb/617/gsb_2/Mercury/fastaggregation.go#L150)）内部已正确调用 `repairAfterLazy`，不会遗漏。风险在于直接使用未导出的 `lazyOR`/`lazyIOR` 函数。

---

## 四、二进制序列化格式

### 4.1 格式总览

该库的二进制序列化格式遵循 [RoaringFormatSpec](https://github.com/RoaringBitmap/RoaringFormatSpec)。核心序列化代码位于 [roaringarray.go:493-567](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L493)（`writeTo`）和 [roaringarray.go:577-703](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L577)（`readFrom`）。

### 4.2 Cookie 的作用

Cookie 是序列化头部的魔数，用于：
1. **格式验证**: 反序列化时校验 cookie 值，若不匹配则报错（[roaringarray.go:611](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L611)）。
2. **区分是否含 run 容器**:

| Cookie 值 | 常量 | 含义 |
|-----------|------|------|
| `12347` | `serialCookie` | 包含 run 容器 |
| `12346` | `serialCookieNoRunContainer` | 不含 run 容器 |

当 cookie 的低 16 位等于 `serialCookie`（12347）时，表示存在 run 容器，此时 cookie 的高 16 位编码了容器数量减 1（[roaringarray.go:597](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L597)）：

```go
if cookie&0x0000FFFF == serialCookie {
    size = cookie>>16 + 1
```

当 cookie 完整等于 `serialCookieNoRunContainer`（12346）时，容器数量在下一个 uint32 中（[roaringarray.go:605](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L605)）。

### 4.3 是否含 run 容器的标志位

当 cookie 表明存在 run 容器时，紧跟 cookie 之后的是一个 **isRun 位图**（[roaringarray.go:599-601](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L599)）：

```go
isRunBitmapSize := (int(size) + 7) / 8
isRunBitmap, err = stream.Next(isRunBitmapSize)
```

这个位图的每一位对应一个容器：若第 i 位置 1，则第 i 个容器是 runContainer16。写入时（[roaringarray.go:515-519](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L515)）：

```go
for i, c := range ra.containers {
    switch c.(type) {
    case *runContainer16:
        runbitmapslice[i/8] |= 1 << (uint(i) % 8)
    }
}
```

### 4.4 容器目录与基数信息的布局

**头部布局（含 run 容器时）**:

```
+-------------------+-------------------+-------------------+-------------------+
| cookie (4 bytes)  | descriptive header| isRun bitmap      | offset header     |
| 低16位=12347      | (4*size bytes)    | ((size+7)/8 bytes)| (4*size bytes,   |
| 高16位=size-1     | key1,card-1, ...  |                   |  条件存在)        |
+-------------------+-------------------+-------------------+-------------------+
```

**头部布局（不含 run 容器时）**:

```
+-------------------+-------------------+-------------------+-------------------+
| cookie (4 bytes)  | size (4 bytes)    | descriptive header| offset header     |
| = 12346           | = numContainers   | (4*size bytes)    | (4*size bytes)    |
+-------------------+-------------------+-------------------+-------------------+
```

**descriptive header** 中每个容器占 4 字节：
- 2 字节：key（uint16 小端）
- 2 字节：cardinality - 1（uint16 小端）

写入代码（[roaringarray.go:530-536](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L530)）：
```go
for i, key := range ra.keys {
    binary.LittleEndian.PutUint16(buf[nw:], key)
    nw += 2
    c := ra.containers[i]
    binary.LittleEndian.PutUint16(buf[nw:], uint16(c.getCardinality()-1))
    nw += 2
}
```

**offset header**：每个容器 4 字节（uint32 小端），存储该容器数据在文件中的起始偏移量。但有一个优化——当容器数量 `< noOffsetThreshold`（即 `< 4`）且含 run 容器时，**省略 offset header**（[roaringarray.go:539](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L539)）：

```go
if !hasRun || (len(ra.keys) >= noOffsetThreshold) {
    // 写 offset header
}
```

### 4.5 大小端处理

- 该库的序列化格式**仅支持小端**（Little Endian）。
- 序列化代码使用 `binary.LittleEndian.PutUint16` / `PutUint32` 写入。
- 反序列化代码使用 `binary.LittleEndian.Uint32` 读取 cookie。
- 对于大端架构，序列化仍会写出小端格式的数据（因为显式使用了 `binary.LittleEndian`），但**零拷贝反序列化**（`FromBuffer` / `FrozenView`）使用 `unsafe.Pointer` 直接将字节切片映射为 uint16/uint64 切片，这在**大端机器上会产生错误结果**。`FrozenView` 明确检查大端 cookie 并返回错误（[serialization_littleendian.go:258-259](file:///e:/gsb/617/gsb_2/Mercury/serialization_littleendian.go#L258)）。

### 4.6 运行时无 run 容器的情况

当 `hasRunCompression()` 返回 `false` 时（[roaringarray.go:705](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L705)），使用 `serialCookieNoRunContainer`（12346）作为 cookie，**不写 isRun 位图**，节省了 `(size+7)/8` 字节。反序列化时根据 cookie 值判断格式。

---

## 五、容易被使用者忽视的潜在风险与陷阱

### 陷阱一：COW 下的共享别名——FromBuffer 后的隐式耦合

**风险描述**: 通过 `FromBuffer` 或 `FrozenView` 构建的位图，其容器直接引用底层 `[]byte` 的内存。所有容器的 `needCopyOnWrite` 被置为 `true`。如果用户在衍生操作（如 `Clone()`、`Or()` 等）后，**未对所有衍生位图调用 `CloneCopyOnWriteContainers`** 就释放了底层 buffer，则所有位图的容器将指向已释放/重用的内存。

**代码依据**:

1. `FromBuffer` 反序列化时设置 `willNeedCopyOnWrite`（[roaringarray.go:592](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L592)）：
   ```go
   willNeedCopyOnWrite := !stream.NextReturnsSafeSlice()
   ```
   并将所有容器的 `needCopyOnWrite` 置为 `true`（[roaringarray.go:655](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L655)）。

2. `FrozenView` 同样将所有 `needCopyOnWrite` 置为 `true`（[serialization_littleendian.go:349](file:///e:/gsb/617/gsb_2/Mercury/serialization_littleendian.go#L349)），并设置 `copyOnWrite = true`。

3. `Clone()` 在 COW 模式下**不深拷贝容器**（[roaringarray.go:263-271](file:///e:/gsb/617/gsb_2/Mercury/roaringarray.go#L263)），只是共享引用并标记 COW。

4. 文档注释（[roaring.go:2179-2189](file:///e:/gsb/617/gsb_2/Mercury/roaring.go#L2179)）明确警告：
   > "you should call CloneCopyOnWriteContainers on all bitmaps that were derived from the 'FromBuffer' bitmap since they may have dependencies on the buf array as well."

**实际后果**: 如果 buffer 来自 `mmap`，在 `munmap` 后访问衍生位图会导致 SIGSEGV；如果 buffer 来自 Go 堆，GC 回收后访问会导致未定义行为。

### 陷阱二：ParOr / 并行聚合对输入的前置假设——非 COW 安全

**风险描述**: `ParOr`（[parallel.go:337](file:///e:/gsb/617/gsb_2/Mercury/parallel.go#L337)）和 `ParHeapOr`（[parallel.go:184](file:///e:/gsb/617/gsb_2/Mercury/parallel.go#L184)）在并行 OR 操作中，多个 goroutine 会**同时读取**输入位图的容器。如果输入位图启用了 COW 且容器被共享，并行读取本身是安全的。但如果输入位图在 `ParOr` 执行期间被其他 goroutine 修改，则存在数据竞争。

更隐蔽的是 `ParOr` 中 `lazyIOrOnRange`（[parallel.go:554](file:///e:/gsb/617/gsb_2/Mercury/parallel.go#L554)）的**就地修改**行为：

```go
func lazyIOrOnRange(ra1, ra2 *roaringArray, start, last uint16) *roaringArray {
    // ...
    c1 := ra1.getFastContainerAtIndex(idx1, true)
    ra1.containers[idx1] = c1.lazyIOR(ra2.getContainerAtIndex(idx2))
    ra1.needCopyOnWrite[idx1] = false
    // ...
    ra1.insertNewKeyValueAt(idx1, key2, ra2.getContainerAtIndex(idx2))
    ra1.needCopyOnWrite[idx1] = true
    // ...
}
```

这里 `ra1`（第一个输入位图的 `roaringArray`）会被**就地修改**——插入新容器、替换现有容器。这意味着：

1. **第一个输入位图会被破坏**: `ParOr` 的第一个输入 bitmap 在操作后不再保持原始状态。
2. **COW 标志被修改**: `needCopyOnWrite` 被设置为 `true` 或 `false`，可能影响后续 COW 行为。
3. **如果第一个输入位图同时被其他 goroutine 访问**，则存在数据竞争。

**代码依据**: `ParOr` 调用链为 `lazyOrOnRange` → `lazyIOrOnRange`，后者直接修改 `ra1` 的 `containers` 和 `needCopyOnWrite` 切片。

### 陷阱三：bitmapContainer.cardinality 的缓存一致性假设

**风险描述**: `bitmapContainer` 的 `cardinality` 字段是一个**缓存值**，在正常操作路径中由库内部维护一致性。但在以下场景中可能不一致：

1. **lazy 操作后未 repair**: 如上文第三节所述，`cardinality` 被设为 `-1`，若忘记 repair 则所有依赖 cardinality 的操作返回错误结果。
2. **通过 `getContainerAtIndex` 获取引用后外部修改**: 如果调用方通过 `getContainerAtIndex` 获取 bitmapContainer 引用，然后直接调用 `iadd`/`iremove` 等方法修改，cardinality 会被正确更新。但如果调用方通过其他方式（如反射、unsafe）修改了 `bitmap` 字段而不更新 `cardinality`，则缓存不一致。
3. **`validate()` 方法会检测不一致**: `bitmapContainer.validate()`（[bitmapcontainer.go:1516-1533](file:///e:/gsb/617/gsb_2/Mercury/bitmapcontainer.go#L1516)）会检查 `bc.cardinality != int(popcntSlice(bc.bitmap))`，但这是一个可选的调试手段。

**代码依据**: `bitmapContainer` 的 `iadd` 方法（[bitmapcontainer.go:375-383](file:///e:/gsb/617/gsb_2/Mercury/bitmapcontainer.go#L375)）通过位运算增量更新 cardinality：

```go
bc.cardinality += int((previous ^ newb) >> (uint(x) % 64))
```

这是一种**分支无关**（branchless）的增量更新，依赖 `previous ^ newb` 的结果：若 bit 已设置则 `previous ^ newb = 0`，cardinality 不变；若 bit 未设置则 `previous ^ newb = 1 << (x%64)`，右移后为 1，cardinality 加 1。

### 陷阱四：序列化跨架构兼容性——大端系统上的零拷贝反序列化

**风险描述**: `FromBuffer` 和 `FrozenView` 使用 `unsafe.Pointer` 将 `[]byte` 直接映射为 `[]uint16` / `[]uint64` / `[]interval16`（[serialization_littleendian.go:66-100](file:///e:/gsb/617/gsb_2/Mercury/serialization_littleendian.go#L66)）。这在**大端系统**上会产生字节序反转，导致数据完全错误。

`FrozenView` 已做了大端检测（[serialization_littleendian.go:257-259](file:///e:/gsb/617/gsb_2/Mercury/serialization_littleendian.go#L257)）：
```go
headerBE := binary.BigEndian.Uint32(buf[len(buf)-4:])
if headerBE&0x7fff == frozenCookie {
    return ErrFrozenBitmapBigEndian
}
```

但 `FromBuffer`（通过 `readFrom`）**没有大端检测**。虽然 `serialization_littleendian.go` 的构建标签限制了只在 little-endian 架构上编译，但如果有人将该文件移植到不支持的平台，就会产生静默的数据损坏。

---

## 附录：关键常量速查表

| 常量 | 值 | 文件位置 | 含义 |
|------|---|---------|------|
| `arrayDefaultMaxSize` | 4096 | [util.go:11](file:///e:/gsb/617/gsb_2/Mercury/util.go#L11) | array 容器最大基数阈值 |
| `arrayLazyLowerBound` | 1024 | [util.go:12](file:///e:/gsb/617/gsb_2/Mercury/util.go#L12) | lazy OR 中提前转为 bitmap 的下限 |
| `maxCapacity` | 65536 | [util.go:13](file:///e:/gsb/617/gsb_2/Mercury/util.go#L13) | 单容器全域基数 |
| `serialCookie` | 12347 | [util.go:16](file:///e:/gsb/617/gsb_2/Mercury/util.go#L16) | 含 run 容器的 cookie |
| `serialCookieNoRunContainer` | 12346 | [util.go:14](file:///e:/gsb/617/gsb_2/Mercury/util.go#L14) | 不含 run 容器的 cookie |
| `invalidCardinality` | -1 | [util.go:15](file:///e:/gsb/617/gsb_2/Mercury/util.go#L15) | lazy 阶段 cardinality 标记 |
| `noOffsetThreshold` | 4 | [util.go:17](file:///e:/gsb/617/gsb_2/Mercury/util.go#L17) | 省略 offset header 的容器数阈值 |
| `MaxUint16` | 65535 | [util.go:28](file:///e:/gsb/617/gsb_2/Mercury/util.go#L28) | 16 位无符号最大值 |
| `frozenCookie` | 13766 | [serialization_littleendian.go:233](file:///e:/gsb/617/gsb_2/Mercury/serialization_littleendian.go#L233) | CRoaring 冻结格式 cookie |
