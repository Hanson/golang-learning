# internal/poll/fd

`fd` 也就是文件描述符，这节我们通过 `unix` 和 `windows` 来了解各自的实现细节。

## unix

先来讲讲 `unix` 下 `fd` 的结构，字段含义已经帮大家翻译出来了。

```go
// fd_unix.go
// FD is a file descriptor. The net and os packages use this type as a
// field of a larger type representing a network connection or OS file.
type FD struct {
	// Lock sysfd and serialize access to Read and Write methods.
	// Sysfd 以及 FD 读写锁
	fdmu fdMutex
	
	// System file descriptor. Immutable until Close.
	// 系统文件描述符。不可变直到关闭。
	Sysfd int

	// I/O poller.
	pd pollDesc

	// Writev cache.
	iovecs *[]syscall.Iovec

	// Semaphore signaled when file is closed.
	// 当文件关闭时发出信号
	csema uint32

	// Non-zero if this file has been set to blocking mode.
	// 如果此文件已设置为阻塞模式，则不为零
	isBlocking uint32

	// Whether this is a streaming descriptor, as opposed to a
	// packet-based descriptor like a UDP socket. Immutable.
	// 这是否是一个流描述符，而不是UDP套接字之类的基于包的描述符。不可变的。
	IsStream bool

	// Whether a zero byte read indicates EOF. This is false for a
	// message based socket connection.
	// 读取0字节是否为EOF。对于基于消息的套接字连接，此值为false。
	ZeroReadIsEOF bool

	// Whether this is a file rather than a network socket.
	// 这是否是一个文件而不是一个网络套接字。
	isFile bool
}

// fd_poll_runtime.go
type pollDesc struct {
    runtimeCtx uintptr
}

// ztypes_linux_amd64.go
type Iovec struct {
    Base *byte
    Len  uint64
}
```