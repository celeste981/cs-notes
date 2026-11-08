# 每周面试知识汇总：2026-11-02 至 2026-11-08

> 本周通用技术知识整理：Go map 并发访问、锁的保护范围与 defer 解锁。重复卡片按主题合并，示例使用通用数据。

## 本周主题

- 什么时候可以并发读 map，哪些操作属于更新。
- 为什么相关访问要使用同一套同步规则，复合操作为什么需要整体保护。
- 用一个 Mutex 保护 Add/Get，并用 defer 安排解锁。

## 主题一：map 的“只读”要检查所有访问路径

### 先讲人话

map 像一本共用字典。判断它是否只读，要看所有使用者：请求只查数据，后台却在删除旧数据，就不属于始终只读的场景。

### 再讲原理

`map` 是按 key 查 value 的容器，`goroutine` 是 Go 的并发执行单元。初始化完成、安全发布且后续不再更新时，可以并发查询和只读遍历；赋值与删除都属于更新，存在并发更新时要协调相关访问。[Go 官方 FAQ](https://go.dev/doc/faq#atomic_maps)

| 操作 | 检查重点 |
| --- | --- |
| 查询、只读的 for range | 确认不存在其他并发更新路径 |
| 给 key 赋值、删除 key | 纳入共享 map 的同步规则 |
| 后台刷新、定时清理 | 与请求路径一起检查 |

### 面试怎么说

> 我会检查 map 的全部访问路径，包括请求处理与后台任务。始终只读的 map 可以并发读取；存在并发更新时，需要统一协调访问。

### 可展开追问

1. **只有后台删除 key，还算只读吗？** 不算，删除也是更新。
2. **两个 goroutine 写不同 key，就能直接并发写吗？** 不能据此认定安全，它们仍在更新同一个内置 map。

### 常见误区

只看当前函数是否写入，漏掉后台刷新或清理任务。

### 公开来源

- [Go FAQ：为什么 map 操作不自动提供原子性](https://go.dev/doc/faq#atomic_maps)

## 主题二：单次访问安全与完整操作是两回事

### 先讲人话

一件事若分两次取得钥匙，中间就可能有人插进来。需要保护“加一”时，要把读、加一、写回当作整件事。

### 再讲原理

`Mutex` 是互斥锁；`RWMutex` 是读写锁，可允许多个读者或一个写者。采用加锁方案时，相关访问遵守同一个锁实例的规则。读写锁的读路径用 `RLock/RUnlock`，写路径用 `Lock/Unlock`。[sync 官方文档](https://pkg.go.dev/sync)

以下是由互斥规则推导的通用例子：初值为 5，即使 Get、Set 各自正确加锁，两个请求仍可能都先读到 5，再各自写回 6；结果是 6，而不是两次加一后的 7。要使这次加一作为整体执行，可由 Add 方法用同一次持锁覆盖读、改、写。

### 面试怎么说

> 我会先确定需要保护的完整操作。共享 map 的相关读写应遵守同一个同步协议；读改写操作需要整体保护，不能只保证 Get 和 Set 各自安全。

### 可展开追问

1. **读写分别使用两把不相关的锁，可以吗？** 两把锁无法相互协调，不能靠这种方式保护同一个共享 map。
2. **写路径加锁、读路径完全不加锁，可以吗？** 读路径可能仍与更新同时发生，要让相关访问共同遵守同步协议。

### 常见误区

把“每次读取与写入分别加锁”当作“整个业务操作不可被其他操作插入”。

### 公开来源

- [sync.Mutex](https://pkg.go.dev/sync#Mutex)
- [sync.RWMutex](https://pkg.go.dev/sync#RWMutex)

## 主题三：一个 Mutex 保护 Add/Get 的最小例子

### 先讲人话

所有读写走同一把锁：先取得，再访问，最后释放。defer 是把释放动作登记为函数返回前执行。

### 再讲原理

下面的示例假设 counts 的访问全部通过 Add/Get 进行。Add 在持锁期间完成整个加一，Get 也使用同一个 mu。`defer mu.Unlock()` 放在 `mu.Lock()` 成功之后，便于让正常返回路径执行解锁。[Effective Go：Defer](https://go.dev/doc/effective_go#defer)

```go
package counter

import "sync"

var mu sync.Mutex
var counts = make(map[string]int)

func Add(key string) {
	mu.Lock()
	defer mu.Unlock()
	counts[key]++
}

func Get(key string) int {
	mu.Lock()
	defer mu.Unlock()
	return counts[key]
}
```

### 面试怎么说

> 这个例子里 Add/Get 使用同一个 Mutex，所有访问都通过它们进行。Add 的读改写整体持锁，Get 也遵守锁规则；defer 把解锁安排在函数返回前。

### 可展开追问

1. **defer Unlock 会立即释放锁吗？** 不会，它登记的是之后执行的调用。
2. **后续增加删除方法，需要做什么？** 让删除也遵守相同同步规则。

### 常见误区

另加一个直接读写 counts 的入口，绕过已有同步规则。

### 公开来源

- [Effective Go：Defer](https://go.dev/doc/effective_go#defer)
- [sync.Mutex](https://pkg.go.dev/sync#Mutex)

## 版本范围

示例使用 Go 1.x 基础 API，不依赖 map 底层布局。讲实现细节时需要注明版本：Go 1.24 引入基于 Swiss Table 的新 map 实现，旧版 hmap/bmap/overflow bucket 的讲法不能直接当成所有版本的实现。[Go 官方说明](https://go.dev/blog/swisstable)
