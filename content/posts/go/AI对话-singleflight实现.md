+++
date = '2025-08-07T17:40:00+08:00'
draft = false
title = 'Go singleflight 源码分析    '
+++

---

**User**

这是什么原因

---

**Cursor**

我需要查看当前打开的文件内容以及相关的错误信息来帮助您诊断问题。让我先检查一下您当前正在查看的文件。

我看到您的光标在第73行，这是一个panic错误处理的位置。让我检查一下是否有任何编译错误或运行时错误信息。

让我检查一下项目的构建状态和可能的错误：

---

**User**

翻一下

---

**Cursor**

我看到您在查看 `singleflight` 包的代码。让我帮您翻译一下您选中的这段注释：

```go
// Do executes and returns the results of the given function, making
// sure that only one execution is in-flight for a given key at a
// time. If a duplicate comes in, the duplicate caller waits for the
// original to complete and receives the same results.
// The return value shared indicates whether v was given to multiple callers.
```

**中文翻译：**

```go
// Do 执行给定的函数并返回结果，确保对于给定的 key
// 在同一时间只有一个执行在进行中。如果有重复的调用进来，
// 重复的调用者会等待原始调用完成并接收相同的结果。
// 返回值 shared 表示 v 是否被提供给了多个调用者。
```

这是 `singleflight` 包中 `Do` 方法的核心功能说明。这个包主要用于防止缓存击穿，确保对于相同的 key，即使有多个并发请求，也只会执行一次实际的操作（比如数据库查询），其他请求会等待并共享这个结果。

```go
// Do executes and returns the results of the given function, making
// sure that only one execution is in-flight for a given key at a
// time. If a duplicate comes in, the duplicate caller waits for the
// original to complete and receives the same results.
// The return value shared indicates whether v was given to multiple callers.
```

```go
// Do 执行给定的函数并返回结果，确保对于给定的 key
// 在同一时间只有一个执行在进行中。如果有重复的调用进来，
// 重复的调用者会等待原始调用完成并接收相同的结果。
// 返回值 shared 表示 v 是否被提供给了多个调用者。
```

---

**User**

翻译这里

---

**Cursor**

我来帮您翻译这段注释：

```go
// The first line of the stack trace is of the form "goroutine N [status]:"
// but by the time the panic reaches Do the goroutine may no longer exist
// and its status will have changed. Trim out the misleading line.
```

**中文翻译：**

```go
// 堆栈跟踪的第一行格式是 "goroutine N [status]:"
// 但是当 panic 到达 Do 方法时，goroutine 可能已经不存在了
// 并且它的状态会发生改变。需要去掉这个误导性的行。
```

这段注释是在解释为什么要处理堆栈跟踪信息。在 Go 语言中，当发生 panic 时会产生堆栈跟踪，但由于并发执行的特性，当 panic 信息传递到 `Do` 方法时，原始的 goroutine 状态可能已经改变，所以需要过滤掉可能误导的第一行信息。

```go
// The first line of the stack trace is of the form "goroutine N [status]:"
// but by the time the panic reaches Do the goroutine may no longer exist
// and its status will have changed. Trim out the misleading line.
```

```go
// 堆栈跟踪的第一行格式是 "goroutine N [status]:"
// 但是当 panic 到达 Do 方法时，goroutine 可能已经不存在了
// 并且它的状态会发生改变。需要去掉这个误导性的行。
```

---

**User**

翻译这里

---

**Cursor**

我来帮您翻译这段注释：

```go
// Ideally, we would wait to take a stack trace until we've determined
// whether this is a panic or a runtime.Goexit.
//
// Unfortunately, the only way we can distinguish the two is to see
// whether the recover stopped the goroutine from terminating, and by
// the time we know that, the part of the stack trace relevant to the
// panic has been discarded.
```

**中文翻译：**

```go
// 理想情况下，我们应该等到确定这是一个 panic 还是 runtime.Goexit 
// 之后再获取堆栈跟踪。
//
// 不幸的是，区分这两者的唯一方法是查看 recover 是否阻止了 
// goroutine 终止，但是当我们知道这一点时，与 panic 相关的
// 堆栈跟踪部分已经被丢弃了。
```

这段注释解释了在处理 panic 和 `runtime.Goexit` 时面临的技术挑战：

- **理想状态**：先判断是 panic 还是 Goexit，再决定是否获取堆栈跟踪
- **实际困难**：只能通过 `recover` 的效果来区分，但此时相关的堆栈信息已经丢失了

`runtime.Goexit` 是 Go 语言中用于终止当前 goroutine 的函数，它不会触发 defer 函数中的 recover，这与 panic 的行为不同。

```go
// Ideally, we would wait to take a stack trace until we've determined
// whether this is a panic or a runtime.Goexit.
//
// Unfortunately, the only way we can distinguish the two is to see
// whether the recover stopped the goroutine from terminating, and by
// the time we know that, the part of the stack trace relevant to the
// panic has been discarded.
```

```go
// 理想情况下，我们应该等到确定这是一个 panic 还是 runtime.Goexit 
// 之后再获取堆栈跟踪。
//
// 不幸的是，区分这两者的唯一方法是查看 recover 是否阻止了 
// goroutine 终止，但是当我们知道这一点时，与 panic 相关的
// 堆栈跟踪部分已经被丢弃了。
```

---

**User**

这里为什么要这么做

---

**Cursor**

这段代码的设计是为了处理一个很特殊的并发场景。让我解释一下为什么要这样做：

## 问题背景

在 `singleflight` 中，当多个 goroutine 同时请求相同的 key 时：
- 第一个 goroutine 执行实际的函数
- 其他 goroutine 通过 channel 等待结果

## 为什么需要特殊处理

```go
// In order to prevent the waiting channels from being blocked forever,
// needs to ensure that this panic cannot be recovered.
// 为了防止等待的 channel 被永久阻塞，
// 需要确保这个 panic 不能被 recover。
```

**关键问题**：如果执行函数的 goroutine 发生了 panic，但这个 panic 被上层的 recover 捕获了，那么：

1. **等待的 goroutine 会永远阻塞** - 因为没有人会向它们的 channel 发送结果
2. **资源泄漏** - 这些 goroutine 永远不会被释放

## 解决方案

```go
if len(c.chans) > 0 {
    go panic(e)  // 在新的 goroutine 中重新 panic
    select {}    // 让当前 goroutine 永远阻塞，保留在崩溃转储中
} else {
    panic(e)     // 没有等待者，直接 panic
}
```

**为什么这样设计**：

1. **`go panic(e)`** - 在新的 goroutine 中重新抛出 panic，这个新的 panic 不会被当前调用栈的 recover 捕获，确保程序会崩溃
2. **`select {}`** - 让当前 goroutine 永远阻塞，这样在程序崩溃时的转储信息中还能看到这个 goroutine 的状态，便于调试
3. **`len(c.chans) > 0`** - 只有当有其他 goroutine 在等待时才需要这种特殊处理

这是一种"宁可崩溃也不要死锁"的设计哲学，确保系统不会因为隐藏的 panic 而进入不可恢复的状态。

```go
// In order to prevent the waiting channels from being blocked forever,
// needs to ensure that this panic cannot be recovered.
// 为了防止等待的 channel 被永久阻塞，
// 需要确保这个 panic 不能被 recover。
```

```go
if len(c.chans) > 0 {
    go panic(e)  // 在新的 goroutine 中重新 panic
    select {}    // 让当前 goroutine 永远阻塞，保留在崩溃转储中
} else {
    panic(e)     // 没有等待者，直接 panic
}
```

---

**User**

 go panic(e)  // 在新的 goroutine 中重新 panic 这一行会造成什么后果？导致主 goroutine 崩溃吗？

---

**Cursor**

很好的问题！让我详细解释 `go panic(e)` 的后果：

## `go panic(e)` 的后果

**是的，会导致整个程序崩溃，包括主 goroutine。**

### 具体过程：

1. **新 goroutine 立即 panic**
   ```go
   go panic(e)  // 启动一个新的 goroutine，这个 goroutine 立即执行 panic(e)
   ```

2. **无法被 recover 捕获**
   - 这个新 goroutine 中的 panic 不在任何 recover 的作用域内
   - 因为它是全新的调用栈，与原来的调用栈完全独立

3. **程序终止**
   - 当一个 goroutine 发生未被 recover 的 panic 时，整个程序会终止
   - 这会影响所有 goroutine，包括主 goroutine

### 为什么要这样设计？

这是一个**故意的设计决策**：

```go
// 伪代码说明意图
if 有其他goroutine在等待 {
    // 情况1：让程序崩溃，避免死锁
    go panic(e)      // 确保程序崩溃
    select {}        // 保持当前goroutine存活，便于调试
} else {
    // 情况2：没有等待者，直接panic
    panic(e)         // 正常的panic处理
}
```

### 设计理念

这体现了 Go 语言中的一个重要原则：**"Don't hide errors that will cause problems later"**

- **如果让 panic 被 recover**：等待的 goroutine 会永远阻塞，造成更严重的问题
- **如果让程序崩溃**：虽然程序终止了，但问题会立即暴露，便于修复

### 实际影响

```go
func main() {
    // 如果 singleflight 中发生了这种情况
    // 整个程序会打印 panic 信息并退出
    // 类似于：
    // panic: some error
    // 
    // goroutine 123 [running]:
    // ...stack trace...
}
```

所以是的，`go panic(e)` 会导致整个程序（包括主 goroutine）崩溃，这是有意为之的设计。

```go
   go panic(e)  // 启动一个新的 goroutine，这个 goroutine 立即执行 panic(e)
```

```go
// 伪代码说明意图
if 有其他goroutine在等待 {
    // 情况1：让程序崩溃，避免死锁
    go panic(e)      // 确保程序崩溃
    select {}        // 保持当前goroutine存活，便于调试
} else {
    // 情况2：没有等待者，直接panic
    panic(e)         // 正常的panic处理
}
```

```go
func main() {
    // 如果 singleflight 中发生了这种情况
    // 整个程序会打印 panic 信息并退出
    // 类似于：
    // panic: some error
    // 
    // goroutine 123 [running]:
    // ...stack trace...
}
```

---

**User**

既然整个程序会崩溃，那这里为什么要 select {}     select {}        // 保持当前goroutine存活，便于调试

---

**Cursor**

很好的观察！这确实看起来有些矛盾。让我解释一下为什么需要 `select {}`：

## 时序问题

关键在于**程序崩溃不是瞬间发生的**，有一个时间窗口：

```go
go panic(e)    // 启动新goroutine，但需要时间调度执行
select {}      // 立即阻塞当前goroutine
```

### 没有 `select {}` 会发生什么：

```go
go panic(e)    // 启动新goroutine
// 函数结束，当前goroutine继续执行或退出
// 可能在新goroutine panic之前就已经清理完毕
```

**问题**：当前 goroutine 可能会：
1. 继续执行后续代码
2. 函数返回
3. goroutine 退出
4. 调用栈信息丢失

### 有了 `select {}` 的效果：

```go
go panic(e)    // 启动新goroutine
select {}      // 当前goroutine永远阻塞在这里
```

**好处**：
1. **保持调用栈**：当前 goroutine 会一直存在，调用栈信息不会丢失
2. **完整的崩溃信息**：程序崩溃时能看到完整的 goroutine 状态
3. **便于调试**：可以看到是从哪里触发的 panic

## 实际的调试价值

### 没有 `select {}`：
```
panic: some error

goroutine 456 [running]:    // 只能看到新goroutine的信息
main.newGoroutine()
    /some/path/file.go:123
```

### 有了 `select {}`：
```
panic: some error

goroutine 456 [running]:    // 新goroutine的panic
main.newGoroutine()
    /some/path/file.go:123

goroutine 123 [select]:     // 原始goroutine还在，能看到完整调用链
golang.org/x/sync/singleflight.(*Group).Do()
    /path/to/singleflight.go:159
main.someFunction()         // 能看到是谁调用了singleflight
    /your/code/file.go:456
```

## 总结

`select {}` 的作用是**在程序崩溃前保持当前 goroutine 的存在**，这样在崩溃转储中就能看到：
- 谁调用了 singleflight
- 完整的调用链
- 更丰富的调试信息

这是一种"临终前保留证据"的做法，虽然程序要崩溃，但要让崩溃信息尽可能有用。

```go
go panic(e)    // 启动新goroutine，但需要时间调度执行
select {}      // 立即阻塞当前goroutine
```

```go
go panic(e)    // 启动新goroutine
// 函数结束，当前goroutine继续执行或退出
// 可能在新goroutine panic之前就已经清理完毕
```

```go
go panic(e)    // 启动新goroutine
select {}      // 当前goroutine永远阻塞在这里
```

```plaintext
panic: some error

goroutine 456 [running]:    // 只能看到新goroutine的信息
main.newGoroutine()
    /some/path/file.go:123
```

```plaintext
panic: some error

goroutine 456 [running]:    // 新goroutine的panic
main.newGoroutine()
    /some/path/file.go:123

goroutine 123 [select]:     // 原始goroutine还在，能看到完整调用链
golang.org/x/sync/singleflight.(*Group).Do()
    /path/to/singleflight.go:159
main.someFunction()         // 能看到是谁调用了singleflight
    /your/code/file.go:456
```

---

**User**

崩溃转储 是什么意思

---

**Cursor**

崩溃转储（Crash Dump）是程序崩溃时生成的详细信息报告。让我详细解释一下：

## 什么是崩溃转储

当 Go 程序发生 panic 时，runtime 会自动生成一份详细的报告，包含：

### 1. **Panic 信息**
```
panic: runtime error: invalid memory address or nil pointer dereference
```

### 2. **调用栈信息（Stack Trace）**
```
goroutine 1 [running]:
main.someFunction(0x0)
    /path/to/your/file.go:42 +0x1a
main.main()
    /path/to/your/file.go:15 +0x2f
```

### 3. **所有 Goroutine 的状态**
```
goroutine 1 [running]:           // 主goroutine正在运行
main.main()
    /path/to/main.go:10

goroutine 2 [select]:            // 另一个goroutine在select等待
golang.org/x/sync/singleflight.(*Group).Do()
    /path/to/singleflight.go:159

goroutine 3 [chan receive]:      // 第三个goroutine在等待channel
main.worker()
    /path/to/worker.go:25
```

## 实际例子

当程序崩溃时，你在终端会看到类似这样的输出：

```bash
$ go run main.go
panic: something went wrong

goroutine 1 [running]:
main.badFunction()
    /Users/cyp/project/main.go:15 +0x95
main.main()
    /Users/cyp/project/main.go:8 +0x25

goroutine 18 [select]:
golang.org/x/sync/singleflight.(*Group).Do(0xc000010180, 0x4b0a20, 0x5, 0x4b7c60, 0x0, 0x0, 0x0)
    /Users/cyp/go/pkg/mod/golang.org/x/sync@v0.1.0/singleflight/singleflight.go:159 +0x3c5
main.getData()
    /Users/cyp/project/cache.go:42 +0x127

exit status 2
```

## 为什么崩溃转储很重要

### 1. **定位问题**
- 能准确看到是在哪一行代码出错
- 完整的函数调用链

### 2. **并发调试**
- 看到所有 goroutine 的状态
- 理解并发执行时的情况

### 3. **生产环境排错**
- 程序崩溃时留下的"遗言"
- 帮助开发者重现和修复 bug

## 在 singleflight 中的意义

回到之前的代码：
```go
if len(c.chans) > 0 {
    go panic(e)    // 新goroutine会出现在崩溃转储中
    select {}      // 当前goroutine也会出现，状态是[select]
} else {
    panic(e)
}
```

**没有 `select {}`**：崩溃转储可能只显示新 goroutine 的信息
**有了 `select {}`**：崩溃转储会显示两个 goroutine，提供更完整的调试信息

这就像是程序崩溃时的"黑匣子"，记录了崩溃瞬间的所有重要信息。

```plaintext
panic: runtime error: invalid memory address or nil pointer dereference
```

```plaintext
goroutine 1 [running]:
main.someFunction(0x0)
    /path/to/your/file.go:42 +0x1a
main.main()
    /path/to/your/file.go:15 +0x2f
```

```plaintext
goroutine 1 [running]:           // 主goroutine正在运行
main.main()
    /path/to/main.go:10

goroutine 2 [select]:            // 另一个goroutine在select等待
golang.org/x/sync/singleflight.(*Group).Do()
    /path/to/singleflight.go:159

goroutine 3 [chan receive]:      // 第三个goroutine在等待channel
main.worker()
    /path/to/worker.go:25
```

```shellscript
$ go run main.go
panic: something went wrong

goroutine 1 [running]:
main.badFunction()
    /Users/cyp/project/main.go:15 +0x95
main.main()
    /Users/cyp/project/main.go:8 +0x25

goroutine 18 [select]:
golang.org/x/sync/singleflight.(*Group).Do(0xc000010180, 0x4b0a20, 0x5, 0x4b7c60, 0x0, 0x0, 0x0)
    /Users/cyp/go/pkg/mod/golang.org/x/sync@v0.1.0/singleflight/singleflight.go:159 +0x3c5
main.getData()
    /Users/cyp/project/cache.go:42 +0x127

exit status 2
```

```go
if len(c.chans) > 0 {
    go panic(e)    // 新goroutine会出现在崩溃转储中
    select {}      // 当前goroutine也会出现，状态是[select]
} else {
    panic(e)
}
```

---

**User**

主进程崩溃退出后，所有gorutine也都退出是怎么实现的

---

**Cursor**

很好的问题！这涉及到 Go runtime 的进程管理机制。让我详细解释一下：

## Go Runtime 的进程模型

### 1. **进程级别的管理**
Go 程序本质上是一个**操作系统进程**，所有 goroutine 都运行在这个进程内：

```
┌─────────────────────────────────────┐
│           Go 进程                    │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐│
│  │goroutine│  │goroutine│  │goroutine││
│  │    1    │  │    2    │  │    3    ││
│  └─────────┘  └─────────┘  └─────────┘│
│                                       │
│         Go Runtime                    │
└─────────────────────────────────────────┘
```

### 2. **Panic 处理机制**

当任何一个 goroutine 发生**未被 recover 的 panic** 时：

```go
// 伪代码展示 runtime 的处理逻辑
func handlePanic(g *goroutine, panicValue interface{}) {
    if !recovered {
        // 1. 打印 panic 信息和堆栈
        printPanicInfo(panicValue)
        printStackTrace(g)
        
        // 2. 打印所有 goroutine 状态
        printAllGoroutines()
        
        // 3. 终止整个进程
        os.Exit(2)  // 进程退出
    }
}
```

### 3. **进程退出的底层实现**

```go
// runtime/panic.go (简化版)
func fatalpanic(msgs *_panic) {
    // 打印 panic 信息
    printpanics(msgs)
    
    // 如果是主 goroutine panic，或者没有其他用户 goroutine
    if islockdowned || atomic.Load(&panicking) > 1 {
        // 直接退出进程
        exit(2)
    }
    
    // 否则也会最终导致进程退出
    systemstack(func() {
        exit(2)
    })
}
```

## 为什么所有 Goroutine 都会退出

### 1. **操作系统层面**
```bash
# 当 Go 进程调用 exit(2) 时
kill -9 <pid>  # 操作系统强制终止整个进程
```

所有属于这个进程的线程（包括运行 goroutine 的 OS 线程）都会被操作系统回收。

### 2. **Go Runtime 层面**
```go
// 当 panic 发生时，runtime 会：
func main() {
    defer func() {
        if r := recover(); r == nil {
            // 如果没有 recover，runtime 会：
            // 1. 停止调度器
            // 2. 停止垃圾回收器  
            // 3. 调用 os.Exit(2)
        }
    }()
}
```

### 3. **实际的执行流程**

```go
func demonstratePanic() {
    go func() {
        fmt.Println("Goroutine 1 开始")
        time.Sleep(10 * time.Second)  // 这个永远不会完成
        fmt.Println("Goroutine 1 结束") // 这行不会执行
    }()
    
    go func() {
        fmt.Println("Goroutine 2 开始")
        panic("崩溃了!")  // 这里 panic
    }()
    
    time.Sleep(5 * time.Second)
    fmt.Println("主函数结束")  // 这行也不会执行
}
```

**输出结果**：
```
Goroutine 1 开始
Goroutine 2 开始
panic: 崩溃了!

goroutine 6 [running]:
main.demonstratePanic.func2()
    /path/to/file.go:XX +0x39
created by main.demonstratePanic
    /path/to/file.go:XX +0x5a

exit status 2
```

## 特殊情况

### 1. **被 recover 捕获的 panic**
```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("捕获了 panic:", r)
            // 程序继续运行，其他 goroutine 不受影响
        }
    }()
    panic("这个 panic 被捕获了")
}()
```

### 2. **主 goroutine 退出**
```go
func main() {
    go func() {
        time.Sleep(10 * time.Second)
        fmt.Println("这行不会打印")
    }()
    
    fmt.Println("主函数结束")
    // main 函数结束，整个程序退出，其他 goroutine 也会被终止
}
```

## 总结

Go 的设计哲学是：**一个 goroutine 的未处理 panic 会导致整个程序崩溃**，这是通过：

1. **Runtime 检测**：检测到未被 recover 的 panic
2. **进程终止**：调用 `os.Exit(2)` 终止整个进程
3. **操作系统回收**：OS 回收进程的所有资源，包括所有线程和 goroutine

这确保了程序不会因为一个 goroutine 的错误而进入不可预知的状态。

```plaintext
┌─────────────────────────────────────┐
│           Go 进程                    │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐│
│  │goroutine│  │goroutine│  │goroutine││
│  │    1    │  │    2    │  │    3    ││
│  └─────────┘  └─────────┘  └─────────┘│
│                                       │
│         Go Runtime                    │
└─────────────────────────────────────────┘
```

```go
// 伪代码展示 runtime 的处理逻辑
func handlePanic(g *goroutine, panicValue interface{}) {
    if !recovered {
        // 1. 打印 panic 信息和堆栈
        printPanicInfo(panicValue)
        printStackTrace(g)
        
        // 2. 打印所有 goroutine 状态
        printAllGoroutines()
        
        // 3. 终止整个进程
        os.Exit(2)  // 进程退出
    }
}
```

```go
// runtime/panic.go (简化版)
func fatalpanic(msgs *_panic) {
    // 打印 panic 信息
    printpanics(msgs)
    
    // 如果是主 goroutine panic，或者没有其他用户 goroutine
    if islockdowned || atomic.Load(&panicking) > 1 {
        // 直接退出进程
        exit(2)
    }
    
    // 否则也会最终导致进程退出
    systemstack(func() {
        exit(2)
    })
}
```

```shellscript
# 当 Go 进程调用 exit(2) 时
kill -9 <pid>  # 操作系统强制终止整个进程
```

```go
// 当 panic 发生时，runtime 会：
func main() {
    defer func() {
        if r := recover(); r == nil {
            // 如果没有 recover，runtime 会：
            // 1. 停止调度器
            // 2. 停止垃圾回收器  
            // 3. 调用 os.Exit(2)
        }
    }()
}
```

```go
func demonstratePanic() {
    go func() {
        fmt.Println("Goroutine 1 开始")
        time.Sleep(10 * time.Second)  // 这个永远不会完成
        fmt.Println("Goroutine 1 结束") // 这行不会执行
    }()
    
    go func() {
        fmt.Println("Goroutine 2 开始")
        panic("崩溃了!")  // 这里 panic
    }()
    
    time.Sleep(5 * time.Second)
    fmt.Println("主函数结束")  // 这行也不会执行
}
```

```plaintext
Goroutine 1 开始
Goroutine 2 开始
panic: 崩溃了!

goroutine 6 [running]:
main.demonstratePanic.func2()
    /path/to/file.go:XX +0x39
created by main.demonstratePanic
    /path/to/file.go:XX +0x5a

exit status 2
```

```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("捕获了 panic:", r)
            // 程序继续运行，其他 goroutine 不受影响
        }
    }()
    panic("这个 panic 被捕获了")
}()
```

```go
func main() {
    go func() {
        time.Sleep(10 * time.Second)
        fmt.Println("这行不会打印")
    }()
    
    fmt.Println("主函数结束")
    // main 函数结束，整个程序退出，其他 goroutine 也会被终止
}
```

---

**User**

对于 singleflight，在实际使用时有什么最佳事件？比如声明一个全局的 singleflight 对象？

---

**Cursor**

很好的问题！`singleflight` 的最佳实践确实很重要。让我详细介绍一下：

## 1. 全局 vs 局部声明

### ✅ **推荐：全局或包级别声明**

```go
package cache

import "golang.org/x/sync/singleflight"

// 全局声明，整个包共享
var g singleflight.Group

func GetUserInfo(userID string) (*User, error) {
    key := fmt.Sprintf("user:%s", userID)
    
    v, err, _ := g.Do(key, func() (interface{}, error) {
        // 实际的数据库查询或API调用
        return fetchUserFromDB(userID)
    })
    
    if err != nil {
        return nil, err
    }
    return v.(*User), nil
}
```

### ❌ **不推荐：函数内局部声明**

```go
func GetUserInfo(userID string) (*User, error) {
    var g singleflight.Group  // 每次调用都创建新的，失去了去重效果
    // ...
}
```

## 2. 实际项目中的最佳实践

### **方案一：按业务模块分组**

```go
package service

import "golang.org/x/sync/singleflight"

type UserService struct {
    sfGroup singleflight.Group  // 每个服务有自己的 singleflight
}

func (s *UserService) GetUser(id string) (*User, error) {
    key := fmt.Sprintf("user:%s", id)
    v, err, _ := s.sfGroup.Do(key, func() (interface{}, error) {
        return s.fetchUserFromDB(id)
    })
    return v.(*User), err
}

type OrderService struct {
    sfGroup singleflight.Group  // 订单服务独立的 singleflight
}
```

### **方案二：全局缓存管理器**

```go
package cache

import "golang.org/x/sync/singleflight"

type CacheManager struct {
    userGroup  singleflight.Group
    orderGroup singleflight.Group
    // 按业务类型分组，避免不同业务的key冲突
}

var DefaultCacheManager = &CacheManager{}

func GetUser(id string) (*User, error) {
    v, err, _ := DefaultCacheManager.userGroup.Do(id, func() (interface{}, error) {
        return fetchUserFromDB(id)
    })
    return v.(*User), err
}
```

## 3. 在您的项目中的应用

基于您的 `baseCache` 项目结构，推荐这样使用：

```go
package cache

import "golang.org/x/sync/singleflight"

// 全局声明，整个包共享
var g singleflight.Group

func GetUserInfo(userID string) (*User, error) {
    key := fmt.Sprintf("user:%s", userID)
    
    v, err, _ := g.Do(key, func() (interface{}, error) {
        // 实际的数据库查询或API调用
        return fetchUserFromDB(userID)
    })
    
    if err != nil {
        return nil, err
    }
    return v.(*User), nil
}
```

```go
func GetUserInfo(userID string) (*User, error) {
    var g singleflight.Group  // 每次调用都创建新的，失去了去重效果
    // ...
}
```

```go
package service

import "golang.org/x/sync/singleflight"

type UserService struct {
    sfGroup singleflight.Group  // 每个服务有自己的 singleflight
}

func (s *UserService) GetUser(id string) (*User, error) {
    key := fmt.Sprintf("user:%s", id)
    v, err, _ := s.sfGroup.Do(key, func() (interface{}, error) {
        return s.fetchUserFromDB(id)
    })
    return v.(*User), err
}

type OrderService struct {
    sfGroup singleflight.Group  // 订单服务独立的 singleflight
}
```

```go
package cache

import "golang.org/x/sync/singleflight"

type CacheManager struct {
    userGroup  singleflight.Group
    orderGroup singleflight.Group
    // 按业务类型分组，避免不同业务的key冲突
}

var DefaultCacheManager = &CacheManager{}

func GetUser(id string) (*User, error) {
    v, err, _ := DefaultCacheManager.userGroup.Do(id, func() (interface{}, error) {
        return fetchUserFromDB(id)
    })
    return v.(*User), err
}
```

基于您的项目结构，推荐这样改进：

### **推荐实现方案**

```go
// app/common/singleflight.go
package common

import "golang.org/x/sync/singleflight"

// 全局的 singleflight 分组管理
var (
    ClassroomSF  singleflight.Group
    UserCenterSF singleflight.Group  
    WalletSF     singleflight.Group
    VideoSF      singleflight.Group
    // 按业务模块分组
)
```

```go
// 在 app/biz/classroom/repository/redisRepo/classroom.go 中使用
package redisRepo

import (
    "baseCache/app/common"
    "golang.org/x/sync/singleflight"
)

func (r *ClassRoomRepo) GetPlanInfo(ctx context.Context, planId int) (*entity.PlanInfo, error) {
    key := fmt.Sprintf("plan_info_%d", planId)
    
    // 使用 singleflight 防止缓存击穿
    v, err, shared := common.ClassroomSF.Do(key, func() (interface{}, error) {
        // 先查 Redis
        if cached := r.getPlanInfoFromRedis(ctx, planId); cached != nil {
            return cached, nil
        }
        
        // Redis 没有，查数据库并写入 Redis
        plan, err := r.getPlanInfoFromDB(ctx, planId)
        if err != nil {
            return nil, err
        }
        
        r.setPlanInfoToRedis(ctx, planId, plan)
        return plan, nil
    })
    
    if err != nil {
        return nil, err
    }
    
    // 可以记录是否是共享结果，用于监控
    if shared {
        logger.Info("singleflight shared result", "key", key)
    }
    
    return v.(*entity.PlanInfo), nil
}
```

## 4. 高级最佳实践

### **带超时控制**

```go
func GetUserWithTimeout(userID string) (*User, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    key := fmt.Sprintf("user:%s", userID)
    
    // 使用 channel 实现超时
    type result struct {
        user *User
        err  error
    }
    
    ch := make(chan result, 1)
    
    go func() {
        v, err, _ := g.Do(key, func() (interface{}, error) {
            return fetchUserFromDB(userID)
        })
        
        if err != nil {
            ch <- result{nil, err}
            return
        }
        ch <- result{v.(*User), nil}
    }()
    
    select {
    case res := <-ch:
        return res.user, res.err
    case <-ctx.Done():
        return nil, ctx.Err()
    }
}
```

### **监控和指标**

```go
func GetUserWithMetrics(userID string) (*User, error) {
    key := fmt.Sprintf("user:%s", userID)
    start := time.Now()
    
    v, err, shared := g.Do(key, func() (interface{}, error) {
        return fetchUserFromDB(userID)
    })
    
    // 记录指标
    duration := time.Since(start)
    if shared {
        metrics.SingleflightHit.Inc()
    } else {
        metrics.SingleflightMiss.Inc()
    }
    metrics.SingleflightDuration.Observe(duration.Seconds())
    
    return v.(*User), err
}
```

## 5. 注意事项

### **Key 的设计**
```go
// ✅ 好的 key 设计
key := fmt.Sprintf("user:info:%d", userID)
key := fmt.Sprintf("order:detail:%s", orderID)

// ❌ 避免 key 冲突
key := fmt.Sprintf("%d", id)  // 太简单，可能冲突
```

### **错误处理**
```go
v, err, shared := g.Do(key, func() (interface{}, error) {
    // 区分可重试和不可重试的错误
    data, err := fetchFromDB(id)
    if err != nil {
        if isRetryableError(err) {
            return nil, err  // 返回错误，其他等待者也会收到错误
        }
        // 对于不可重试的错误，可能需要特殊处理
    }
    return data, err
})
```

总结：**推荐使用全局或包级别的 singleflight，按业务模块分组，配合适当的监控和错误处理。**

```go
// app/common/singleflight.go
package common

import "golang.org/x/sync/singleflight"

// 全局的 singleflight 分组管理
var (
    ClassroomSF  singleflight.Group
    UserCenterSF singleflight.Group  
    WalletSF     singleflight.Group
    VideoSF      singleflight.Group
    // 按业务模块分组
)
```

```go
// 在 app/biz/classroom/repository/redisRepo/classroom.go 中使用
package redisRepo

import (
    "baseCache/app/common"
    "golang.org/x/sync/singleflight"
)

func (r *ClassRoomRepo) GetPlanInfo(ctx context.Context, planId int) (*entity.PlanInfo, error) {
    key := fmt.Sprintf("plan_info_%d", planId)
    
    // 使用 singleflight 防止缓存击穿
    v, err, shared := common.ClassroomSF.Do(key, func() (interface{}, error) {
        // 先查 Redis
        if cached := r.getPlanInfoFromRedis(ctx, planId); cached != nil {
            return cached, nil
        }
        
        // Redis 没有，查数据库并写入 Redis
        plan, err := r.getPlanInfoFromDB(ctx, planId)
        if err != nil {
            return nil, err
        }
        
        r.setPlanInfoToRedis(ctx, planId, plan)
        return plan, nil
    })
    
    if err != nil {
        return nil, err
    }
    
    // 可以记录是否是共享结果，用于监控
    if shared {
        logger.Info("singleflight shared result", "key", key)
    }
    
    return v.(*entity.PlanInfo), nil
}
```

```go
func GetUserWithTimeout(userID string) (*User, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    key := fmt.Sprintf("user:%s", userID)
    
    // 使用 channel 实现超时
    type result struct {
        user *User
        err  error
    }
    
    ch := make(chan result, 1)
    
    go func() {
        v, err, _ := g.Do(key, func() (interface{}, error) {
            return fetchUserFromDB(userID)
        })
        
        if err != nil {
            ch <- result{nil, err}
            return
        }
        ch <- result{v.(*User), nil}
    }()
    
    select {
    case res := <-ch:
        return res.user, res.err
    case <-ctx.Done():
        return nil, ctx.Err()
    }
}
```

```go
func GetUserWithMetrics(userID string) (*User, error) {
    key := fmt.Sprintf("user:%s", userID)
    start := time.Now()
    
    v, err, shared := g.Do(key, func() (interface{}, error) {
        return fetchUserFromDB(userID)
    })
    
    // 记录指标
    duration := time.Since(start)
    if shared {
        metrics.SingleflightHit.Inc()
    } else {
        metrics.SingleflightMiss.Inc()
    }
    metrics.SingleflightDuration.Observe(duration.Seconds())
    
    return v.(*User), err
}
```

```go
// app/common/singleflight.go
package common

import "golang.org/x/sync/singleflight"

// 全局的 singleflight 分组管理
var (
    ClassroomSF  singleflight.Group
    UserCenterSF singleflight.Group  
    WalletSF     singleflight.Group
    VideoSF      singleflight.Group
    // 按业务模块分组
)
```

```go
// 在 app/biz/classroom/repository/redisRepo/classroom.go 中使用
package redisRepo

import (
    "baseCache/app/common"
    "golang.org/x/sync/singleflight"
)

func (r *ClassRoomRepo) GetPlanInfo(ctx context.Context, planId int) (*entity.PlanInfo, error) {
    key := fmt.Sprintf("plan_info_%d", planId)
    
    // 使用 singleflight 防止缓存击穿
    v, err, shared := common.ClassroomSF.Do(key, func() (interface{}, error) {
        // 先查 Redis
        if cached := r.getPlanInfoFromRedis(ctx, planId); cached != nil {
            return cached, nil
        }
        
        // Redis 没有，查数据库并写入 Redis
        plan, err := r.getPlanInfoFromDB(ctx, planId)
        if err != nil {
            return nil, err
        }
        
        r.setPlanInfoToRedis(ctx, planId, plan)
        return plan, nil
    })
    
    if err != nil {
        return nil, err
    }
    
    // 可以记录是否是共享结果，用于监控
    if shared {
        logger.Info("singleflight shared result", "key", key)
    }
    
    return v.(*entity.PlanInfo), nil
}
```

```go
func GetUserWithTimeout(userID string) (*User, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    key := fmt.Sprintf("user:%s", userID)
    
    // 使用 channel 实现超时
    type result struct {
        user *User
        err  error
    }
    
    ch := make(chan result, 1)
    
    go func() {
        v, err, _ := g.Do(key, func() (interface{}, error) {
            return fetchUserFromDB(userID)
        })
        
        if err != nil {
            ch <- result{nil, err}
            return
        }
        ch <- result{v.(*User), nil}
    }()
    
    select {
    case res := <-ch:
        return res.user, res.err
    case <-ctx.Done():
        return nil, ctx.Err()
    }
}
```

```go
func GetUserWithMetrics(userID string) (*User, error) {
    key := fmt.Sprintf("user:%s", userID)
    start := time.Now()
    
    v, err, shared := g.Do(key, func() (interface{}, error) {
        return fetchUserFromDB(userID)
    })
    
    // 记录指标
    duration := time.Since(start)
    if shared {
        metrics.SingleflightHit.Inc()
    } else {
        metrics.SingleflightMiss.Inc()
    }
    metrics.SingleflightDuration.Observe(duration.Seconds())
    
    return v.(*User), err
}
```

```go
// ✅ 好的 key 设计
key := fmt.Sprintf("user:info:%d", userID)
key := fmt.Sprintf("order:detail:%s", orderID)

// ❌ 避免 key 冲突
key := fmt.Sprintf("%d", id)  // 太简单，可能冲突
```

```go
v, err, shared := g.Do(key, func() (interface{}, error) {
    // 区分可重试和不可重试的错误
    data, err := fetchFromDB(id)
    if err != nil {
        if isRetryableError(err) {
            return nil, err  // 返回错误，其他等待者也会收到错误
        }
        // 对于不可重试的错误，可能需要特殊处理
    }
    return data, err
})
```

---

**User**

在你的 方案一：按业务模块分组 中，我看到 这种定义 ，那么这种情况在 UserService 是单例实现的时候才有效吧？如果每次请求过来新获取一个 UserService 是不是就没有用了？

---

**Cursor**

非常好的观察！您说得完全正确！这是一个非常重要的细节。

## 问题分析

如果 `UserService` 不是单例，每次请求都创建新实例：

```go
// ❌ 错误的使用方式
func HandleRequest() {
    userService := &UserService{}  // 每次都创建新实例
    user, err := userService.GetUser("123")  // singleflight 失效！
}
```

每个新的 `UserService` 实例都有自己独立的 `sfGroup`，完全失去了 singleflight 的去重效果。

## 正确的实现方式

### **方案一：确保服务是单例**

```go
package service

import "golang.org/x/sync/singleflight"

type UserService struct {
    sfGroup singleflight.Group
}

// 全局单例
var userServiceInstance *UserService
var once sync.Once

func GetUserService() *UserService {
    once.Do(func() {
        userServiceInstance = &UserService{}
    })
    return userServiceInstance
}

// 使用时
func HandleRequest() {
    userService := GetUserService()  // 获取单例
    user, err := userService.GetUser("123")
}
```

### **方案二：将 singleflight 提取到包级别（推荐）**

```go
package service

import "golang.org/x/sync/singleflight"

// 包级别的 singleflight，所有 UserService 实例共享
var userSingleflight singleflight.Group

type UserService struct {
    // 其他字段，不包含 sfGroup
}

func (s *UserService) GetUser(id string) (*User, error) {
    key := fmt.Sprintf("user:%s", id)
    // 使用包级别的 singleflight
    v, err, _ := userSingleflight.Do(key, func() (interface{}, error) {
        return s.fetchUserFromDB(id)
    })
    return v.(*User), err
}

// 这样即使每次创建新的 UserService 实例也没问题
func HandleRequest() {
    userService := &UserService{}  // 新实例
    user, err := userService.GetUser("123")  // singleflight 仍然有效
}
```

### **方案三：依赖注入模式**

```go
package service

import "golang.org/x/sync/singleflight"

type UserService struct {
    sfGroup *singleflight.Group  // 注入的 singleflight
}

func NewUserService(sf *singleflight.Group) *UserService {
    return &UserService{
        sfGroup: sf,
    }
}

// 在应用启动时
func main() {
    // 全局的 singleflight 实例
    userSF := &singleflight.Group{}
    
    // 注入到服务中
    userService := NewUserService(userSF)
    
    // 或者使用依赖注入框架
}
```

## 在您的项目中的建议

基于您的 `baseCache` 项目，我建议采用 **方案二**：

```go
// app/common/singleflight.go
package common

import "golang.org/x/sync/singleflight"

// 按业务模块定义包级别的 singleflight
var (
    ClassroomSF  singleflight.Group
    UserCenterSF singleflight.Group
    WalletSF     singleflight.Group
    VideoSF      singleflight.Group
)
```

```go
// app/biz/classroom/service/service.go
package service

import "baseCache/app/common"

type ClassroomService struct {
    // 不包含 singleflight 字段
}

func (s *ClassroomService) GetPlanInfo(planId int) (*entity.PlanInfo, error) {
    key := fmt.Sprintf("plan_info_%d", planId)
    
    // 使用全局的 ClassroomSF
    v, err, _ := common.ClassroomSF.Do(key, func() (interface{}, error) {
        return s.fetchPlanFromDB(planId)
    })
    
    return v.(*entity.PlanInfo), err
}

// 这样无论如何创建 ClassroomService 实例都没问题
func HandleClassroomRequest() {
    service := &ClassroomService{}  // 新实例也能正确使用 singleflight
    plan, err := service.GetPlanInfo(123)
}
```

## 总结

您的观察完全正确！在使用 singleflight 时：

1. **如果放在 struct 中**：必须确保该 struct 是单例
2. **推荐做法**：将 singleflight 定义在包级别，让所有实例共享
3. **这样即使每次创建新的服务实例，singleflight 的去重效果依然有效**

这是一个很容易被忽视但非常重要的实现细节！

```go
// ❌ 错误的使用方式
func HandleRequest() {
    userService := &UserService{}  // 每次都创建新实例
    user, err := userService.GetUser("123")  // singleflight 失效！
}
```

```go
package service

import "golang.org/x/sync/singleflight"

type UserService struct {
    sfGroup singleflight.Group
}

// 全局单例
var userServiceInstance *UserService
var once sync.Once

func GetUserService() *UserService {
    once.Do(func() {
        userServiceInstance = &UserService{}
    })
    return userServiceInstance
}

// 使用时
func HandleRequest() {
    userService := GetUserService()  // 获取单例
    user, err := userService.GetUser("123")
}
```

```go
package service

import "golang.org/x/sync/singleflight"

// 包级别的 singleflight，所有 UserService 实例共享
var userSingleflight singleflight.Group

type UserService struct {
    // 其他字段，不包含 sfGroup
}

func (s *UserService) GetUser(id string) (*User, error) {
    key := fmt.Sprintf("user:%s", id)
    // 使用包级别的 singleflight
    v, err, _ := userSingleflight.Do(key, func() (interface{}, error) {
        return s.fetchUserFromDB(id)
    })
    return v.(*User), err
}

// 这样即使每次创建新的 UserService 实例也没问题
func HandleRequest() {
    userService := &UserService{}  // 新实例
    user, err := userService.GetUser("123")  // singleflight 仍然有效
}
```

```go
package service

import "golang.org/x/sync/singleflight"

type UserService struct {
    sfGroup *singleflight.Group  // 注入的 singleflight
}

func NewUserService(sf *singleflight.Group) *UserService {
    return &UserService{
        sfGroup: sf,
    }
}

// 在应用启动时
func main() {
    // 全局的 singleflight 实例
    userSF := &singleflight.Group{}
    
    // 注入到服务中
    userService := NewUserService(userSF)
    
    // 或者使用依赖注入框架
}
```

```go
// app/common/singleflight.go
package common

import "golang.org/x/sync/singleflight"

// 按业务模块定义包级别的 singleflight
var (
    ClassroomSF  singleflight.Group
    UserCenterSF singleflight.Group
    WalletSF     singleflight.Group
    VideoSF      singleflight.Group
)
```

```go
// app/biz/classroom/service/service.go
package service

import "baseCache/app/common"

type ClassroomService struct {
    // 不包含 singleflight 字段
}

func (s *ClassroomService) GetPlanInfo(planId int) (*entity.PlanInfo, error) {
    key := fmt.Sprintf("plan_info_%d", planId)
    
    // 使用全局的 ClassroomSF
    v, err, _ := common.ClassroomSF.Do(key, func() (interface{}, error) {
        return s.fetchPlanFromDB(planId)
    })
    
    return v.(*entity.PlanInfo), err
}

// 这样无论如何创建 ClassroomService 实例都没问题
func HandleClassroomRequest() {
    service := &ClassroomService{}  // 新实例也能正确使用 singleflight
    plan, err := service.GetPlanInfo(123)
}
```

---

**User**

在 带超时控制 的实现中，为什么要新启动一个 goroutine？

---

**Cursor**

非常好的问题！这涉及到 `singleflight.Do()` 方法的阻塞特性。让我详细解释一下：

## 问题的核心

`singleflight.Do()` 是一个**阻塞调用**：

```go
// singleflight.Do() 的行为
v, err, shared := g.Do(key, func() (interface{}, error) {
    // 这个函数可能执行很久
    time.Sleep(10 * time.Second)  // 假设数据库查询很慢
    return fetchUserFromDB(userID)
})
// 在函数执行完之前，这里不会被执行
```

## 如果不使用 goroutine 会怎样？

### ❌ **错误的实现**
```go
func GetUserWithTimeout(userID string) (*User, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    key := fmt.Sprintf("user:%s", userID)
    
    // 直接调用，没有 goroutine
    v, err, _ := g.Do(key, func() (interface{}, error) {
        return fetchUserFromDB(userID)  // 假设这需要 10 秒
    })
    
    // 问题：这个 select 永远不会被执行到！
    // 因为上面的 g.Do() 会阻塞 10 秒才返回
    select {
    case <-ctx.Done():  // 这个分支永远不会被触发
        return nil, ctx.Err()
    default:
        return v.(*User), err
    }
}
```

**结果**：超时控制完全失效，函数依然会等待 10 秒。

## 使用 goroutine 的原因

### ✅ **正确的实现**
```go
func GetUserWithTimeout(userID string) (*User, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    key := fmt.Sprintf("user:%s", userID)
    
    ch := make(chan result, 1)
    
    // 在新的 goroutine 中执行阻塞的 singleflight.Do()
    go func() {
        v, err, _ := g.Do(key, func() (interface{}, error) {
            return fetchUserFromDB(userID)  // 10 秒的阻塞操作
        })
        
        // 将结果发送到 channel
        if err != nil {
            ch <- result{nil, err}
        } else {
            ch <- result{v.(*User), nil}
        }
    }()
    
    // 主 goroutine 可以在这里等待两个事件：
    select {
    case res := <-ch:          // singleflight 完成
        return res.user, res.err
    case <-ctx.Done():         // 超时
        return nil, ctx.Err()  // 5 秒后返回超时错误
    }
}
```

## 执行流程对比

### **没有 goroutine 的执行流程**：
```
1. 开始执行 GetUserWithTimeout()
2. 调用 g.Do() -> 阻塞 10 秒
3. 10 秒后 g.Do() 返回
4. 执行 select（但此时已经超过 5 秒了）
5. 返回结果（超时控制失效）
```

### **使用 goroutine 的执行流程**：
```
主 goroutine:                    新 goroutine:
1. 创建 channel                  
2. 启动 goroutine               -> 3. 执行 g.Do()（阻塞 10 秒）
4. 进入 select 等待
5. 5 秒后 ctx.Done() 触发
6. 返回超时错误                 -> （g.Do() 可能还在执行）
```

## 实际的时序图

```
时间轴: 0s -------- 5s -------- 10s
       |           |           |
主线程: 启动 ----> 超时返回      |
       |           |           |
新线程: g.Do()开始 -------- g.Do()完成
       |                       |
       |<------ 阻塞 10 秒 ---->|
```

## 更直观的例子

```go
func demonstrateBlocking() {
    fmt.Println("开始时间:", time.Now())
    
    // 模拟不使用 goroutine
    fmt.Println("调用 singleflight...")
    g.Do("test", func() (interface{}, error) {
        fmt.Println("开始执行慢操作:", time.Now())
        time.Sleep(10 * time.Second)
        fmt.Println("慢操作完成:", time.Now())
        return "result", nil
    })
    
    fmt.Println("singleflight 返回:", time.Now())
    // 输出显示：从开始到这里经过了完整的 10 秒
}
```

## 总结

使用 goroutine 的原因是：

1. **`singleflight.Do()` 是阻塞的**，会等待函数执行完成
2. **如果不使用 goroutine**，超时检查的代码永远不会被执行到
3. **使用 goroutine** 让阻塞操作在后台执行，主线程可以同时监听超时事件
4. **实现真正的超时控制**，而不是假的超时控制

这是并发编程中一个经典的模式：**将阻塞操作放到 goroutine 中，主线程负责协调和超时控制**。

```go
// singleflight.Do() 的行为
v, err, shared := g.Do(key, func() (interface{}, error) {
    // 这个函数可能执行很久
    time.Sleep(10 * time.Second)  // 假设数据库查询很慢
    return fetchUserFromDB(userID)
})
// 在函数执行完之前，这里不会被执行
```

```go
func GetUserWithTimeout(userID string) (*User, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    key := fmt.Sprintf("user:%s", userID)
    
    // 直接调用，没有 goroutine
    v, err, _ := g.Do(key, func() (interface{}, error) {
        return fetchUserFromDB(userID)  // 假设这需要 10 秒
    })
    
    // 问题：这个 select 永远不会被执行到！
    // 因为上面的 g.Do() 会阻塞 10 秒才返回
    select {
    case <-ctx.Done():  // 这个分支永远不会被触发
        return nil, ctx.Err()
    default:
        return v.(*User), err
    }
}
```

```go
func GetUserWithTimeout(userID string) (*User, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    key := fmt.Sprintf("user:%s", userID)
    
    ch := make(chan result, 1)
    
    // 在新的 goroutine 中执行阻塞的 singleflight.Do()
    go func() {
        v, err, _ := g.Do(key, func() (interface{}, error) {
            return fetchUserFromDB(userID)  // 10 秒的阻塞操作
        })
        
        // 将结果发送到 channel
        if err != nil {
            ch <- result{nil, err}
        } else {
            ch <- result{v.(*User), nil}
        }
    }()
    
    // 主 goroutine 可以在这里等待两个事件：
    select {
    case res := <-ch:          // singleflight 完成
        return res.user, res.err
    case <-ctx.Done():         // 超时
        return nil, ctx.Err()  // 5 秒后返回超时错误
    }
}
```

```plaintext
1. 开始执行 GetUserWithTimeout()
2. 调用 g.Do() -> 阻塞 10 秒
3. 10 秒后 g.Do() 返回
4. 执行 select（但此时已经超过 5 秒了）
5. 返回结果（超时控制失效）
```

```plaintext
主 goroutine:                    新 goroutine:
1. 创建 channel                  
2. 启动 goroutine               -> 3. 执行 g.Do()（阻塞 10 秒）
4. 进入 select 等待
5. 5 秒后 ctx.Done() 触发
6. 返回超时错误                 -> （g.Do() 可能还在执行）
```

```plaintext
时间轴: 0s -------- 5s -------- 10s
       |           |           |
主线程: 启动 ----> 超时返回      |
       |           |           |
新线程: g.Do()开始 -------- g.Do()完成
       |                       |
       |<------ 阻塞 10 秒 ---->|
```

```go
func demonstrateBlocking() {
    fmt.Println("开始时间:", time.Now())
    
    // 模拟不使用 goroutine
    fmt.Println("调用 singleflight...")
    g.Do("test", func() (interface{}, error) {
        fmt.Println("开始执行慢操作:", time.Now())
        time.Sleep(10 * time.Second)
        fmt.Println("慢操作完成:", time.Now())
        return "result", nil
    })
    
    fmt.Println("singleflight 返回:", time.Now())
    // 输出显示：从开始到这里经过了完整的 10 秒
}
```

