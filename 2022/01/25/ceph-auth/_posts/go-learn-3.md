---
title: 每天5分钟go-3
date: 2018-10-06 15:49:20
tags: go
---

# go的并发
- 同步时遇到竟态
有两种解决：
1. 原子操作（sync/atomic）
2. 临界区（互斥）

# go中的管道使用
- os/exec
使用示例
```
cmd0 := exec.Command("echo", "-n", "My first")
cmd0.Start()
stdout0, err := cmd0.StdoutPipe()
output0 := make([]byte, 30)
n, err := stdout0.Read(output0)
```
- 两个进程间管道的使用示例
```
cmd1 := exec.Command("ps", "aux")
cmd2 := exec.Command("grep", "apipe")
var outputBuf1 bytes.Buffer
cmd1.Stdout = &outputBuf1
if err := cmd1.Start(); err != nil {
        fmt.Printf("Error: The first command can not be startup %s\n", err)
        return
}
if err := cmd1.Wait(); err != nil {
        fmt.Printf("Error: Couldn't wait for the first command: %s\n", err)
        return
}
cmd2.Stdin = &outputBuf1
var outputBuf2 bytes.Buffer
cmd2.Stdout = &outputBuf2
if err := cmd2.Start(); err != nil {
        fmt.Printf("Error: The second command can not be startup: %s\n", err)
        return
}
if err := cmd2.Wait(); err != nil {
        fmt.Printf("Error: Couldn't wait for the second command: %s\n", err)
        return
}
fmt.Printf("%s\n", outputBuf2.Bytes())
```
此管道为匿名管道
- 命令管道创建
```
reader, writer, err := os.Pipe()
```
注意：匿名管道会在管道缓冲区被写满之后使写数据的进程阻塞，命名管道会在其中一段未被就绪钱阻塞另一端的进程
- 基于内存的有原子操作保证的管道
```
reader, writer := io.Pipe()
```
# 信号
是IPC中唯一一种异步通信方式
信号的来源
```
1. 键盘输入
2. 硬件故障
3. 系统函数调用
4. 软件中的非法运算
```
信号所在代码库os/signal
- 发送信号函数
```
func Notify（）
```
- 两个信号是不能被忽略的
信号可以被忽略，捕捉和执行默认操作，有两个信号不能被忽略:SIGKILL和SIGSTOP
- 恢复信号的默认操作
```
func Stop()
```
- go中操作进程
```
1. os.StartProcess()启动进程，返回进程值
2. os.FindProcess()查找进程， 返回进程值
3. 调用进程值的signal方法可以给进程发送信号
```
# socket
包net
- 服务端
```
net.Listen()
lintener.Accept()
```
- 客户端
```
func Dial()
func DialTime()同时设置超时时间
```
- 操作系统的socket操作不是阻塞的
- socket关闭之后，read会返回io.EOF错误
- net.Conn方法
```
Read
Write
Close
LocalAddr
RemoteAddr
SetDeadline（超时时间，包括读写，还有其他）,超时时间需要每次循环都要设置，使用time.Time{}零值参数取消超时时间
SetReadDeadline（读超时时间）
SetWriteDeadline（写超时时间）
```
注:客户端也是有端口，毕竟是端到端的通信，不过客户端的端口是内核自己分配的
- 闲置连接
超时说明这个连接已经是闲置连接
