---
title: Go底层实现速查
tags:
  - Go
  - 底层
  - 八股
related_project:
  - "[[20-Go-Ecosystem (Go 语言与生态)/Go语言核心]]"
  - "[[20-Go-Ecosystem (Go 语言与生态)/Go八股问答集]]"
---

## 1. interface 底层实现

### 空接口 `interface{}` → eface

```go
type eface struct {
    _type *_type         // 类型信息
    data  unsafe.Pointer // 指向实际数据的指针
}
```

- 任何类型都能赋值给 `interface{}`
- `_type` 描述类型元信息（大小、对齐、GC 位图等）
- `data` 指向堆上或栈上的实际值

### 带方法的 interface → iface

```go
type iface struct {
    tab  *itab           // 方法表
    data unsafe.Pointer  // 指向实际数据的指针
}

type itab struct {
    inter *interfacetype // 接口类型信息（有哪些方法）
    _type *_type         // 具体类型信息
    hash  uint32         // _type.hash 的拷贝，用于快速类型断言
    fun   [1]uintptr     // 方法地址数组（变长），按接口方法声明顺序排列
}
```

- `itab.fun` 是方法指针数组，调用接口方法时通过它做间接调用
- `itab` 会被缓存（全局哈希表 `itabTable`），同一对 (interface, concrete type) 只计算一次

### 高频考点

**interface == nil 的坑**：
```go
var err *MyError = nil
var e error = err  // e != nil！因为 eface 的 _type 不为 nil
```
interface 为 nil 要求 `_type` 和 `data` **都为 nil**。赋值后 `_type` 有值，所以 e != nil。

**值接收者 vs 指针接收者**：
- 值接收者实现接口 → 值和指针都能赋值给接口
- 指针接收者实现接口 → 只有指针能赋值给接口（值没有方法集）

**类型断言和类型 switch**：
- `v, ok := i.(T)` → 检查 `itab._type` 是否匹配
- 底层比较 `itab.hash`，命中快

---

## 2. slice 底层

```go
type SliceHeader struct {
    Data uintptr
    Len  int
    Cap  int
}
```

- 切片是底层数组的一个**视图**，多个 slice 可共享同一数组
- `append` 超出 Cap 时扩容，可能脱离原数组
- `copy(dst, src)` 深拷贝，`=` 赋值浅拷贝

---

## 3. string 底层

```go
type StringHeader struct {
    Data uintptr
    Len  int
}
```

- string 不可变，没有 Cap 字段
- 转 `[]byte` 必须拷贝（保证不可变性）
- 编译器对 `string(b) == "literal"` 和 `m[string(b)]` 做了无拷贝优化

---

## 4. map 底层 hmap

```go
type hmap struct {
    count     int      // 元素个数
    B         uint8    // 桶数 = 2^B
    buckets   unsafe.Pointer
    oldbuckets unsafe.Pointer // 扩容时旧桶
    overflow  *[...]unsafe.Pointer
}
```

- 每个 bucket 存 8 个 kv 对 + overflow 指针
- 哈希低 B 位定位桶，高 8 位做 tophash 快速比较
- 负载因子 > 6.5 触发翻倍扩容
- Go 1.24+ 引入 Swiss Table（开放寻址）

---

## 5. channel 底层 hchan

```go
type hchan struct {
    qcount   uint
    dataqsiz uint
    buf      unsafe.Pointer
    elemsize uint16
    closed   uint32
    elemtype *_type
    sendx    uint
    recvx    uint
    sendq    waitq
    recvq    waitq
    lock     mutex
}
```

- 环形缓冲区 `buf`，`sendx`/`recvx` 环形索引
- `sendq`/`recvq` 是 sudog 链表，存放阻塞的 goroutine
- 关闭 channel 唤醒所有 recvq 中的 G

---

## 6. goroutine 底层 g

```go
type g struct {
    stack       stack    // 栈空间 {lo, hi}
    stackguard0 uintptr  // 栈增长检查点
    sched       gobuf    // 调度上下文（PC, SP, BP）
    goid        int64
    status      uint32   // _Grunnable, _Grunning, _Gwaiting...
    m           *m       // 绑定的 M
    schedlink   guintptr
}
```

- 初始栈 2KB，按需增长（分段栈 → 连续栈）
- `status` 状态机：_Gidle → _Grunnable → _Grunning → _Gwaiting → _Gdead

---

## 7. 内存分配三层次

```
线程缓存 mcache → 中心缓存 mcentral → 堆 mheap
```

- **mcache**：每个 P 一个，小对象（<32KB）无锁分配
- **mcentral**：按 span class 分类，需要加锁
- **mheap**：全局唯一，管理大对象（>32KB）和向 OS 申请内存

对象分三档：
- 微对象 <16B：mcache tiny allocator 合并分配
- 小对象 16B~32KB：mcache 按 size class 分配
- 大对象 >32KB：直接从 mheap 分配

---

## 8. GC 三色标记 + 混合写屏障

流程：
1. STW（短暂）：开启写屏障，扫描栈根对象
2. 并发标记：三色标记法，用户代码和标记并发执行
3. STW（短暂）：重新扫描栈（Go 1.8 前），关闭写屏障
4. 并发清扫：清除白色对象

混合写屏障（Go 1.8+）= Dijkstra 插入屏障 + Yuasa 删除屏障：
- 新指向的对象标灰
- 被覆盖引用的旧对象标灰
- 避免了栈的重新扫描，STW 降到亚毫秒

---

## 9. defer 底层

- Go 1.14 前：defer 存在 `_defer` 链表（堆分配）
- Go 1.14+：开放编码（open-coded）defer，直接在栈上编码，最后一个 defer 不进链表
- panic 时遍历 `_defer` 链表，LIFO 顺序执行

---

## 10. reflect 底层要点

- `reflect.TypeOf()` 返回 `*rtype`，即 `_type` 的别名
- `reflect.ValueOf()` 返回 `Value{typ, ptr, flag}`
- 反射调用方法有额外开销（类型检查 + 间接调用 + 可能逃逸到堆）
- 接口到反射再回接口（`i.(reflect.Value).Interface()`）性能差
