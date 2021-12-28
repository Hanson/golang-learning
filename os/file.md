# os/file

本篇幅我们通过文件的创建、打开、写入、读取来讲讲文件模块。

## file 结构

我们先来看看 file 的结构

```go
package os

// file_unix.go
type file struct {
	pfd         poll.FD
	name        string
	dirinfo     *dirInfo // nil unless directory being read
	nonblock    bool     // whether we set nonblocking mode
	stdoutOrErr bool     // whether this is stdout or stderr
	appendMode  bool     // whether file is opened for appending
}

// file_windows.go
type file struct {
	pfd        poll.FD
	name       string
	dirinfo    *dirInfo // nil unless directory being read
	appendMode bool     // whether file is opened for appending
}
```

因为 `Windows` 与 `unix` 两个操作系统的差异，结构上也会有些稍微的差别

可以看出 `unix` 下多了 `nonblock` 和 `stdoutOrErr` 两个参数

我们先来看看其他公共参数代表了什么。

`pfd` 文件描述符
`name` 文件名
`dirinfo` 当文件为文件夹时的信息
`appendMode` 文件是打开还是 append 模式
`nonblock` 是否非阻塞模式

无论创建还是打开文件，都需要我们制定打开模式以及打开方式

## 打开方式

```go
//打开方式
const (
    //只读模式
    O_RDONLY int = syscall.O_RDONLY // open the file read-only.
    //只写模式
    O_WRONLY int = syscall.O_WRONLY // open the file write-only.
    //可读可写
    O_RDWR int = syscall.O_RDWR // open the file read-write.
    //追加内容
    O_APPEND int = syscall.O_APPEND // append data to the file when writing.
    //创建文件,如果文件不存在
    O_CREATE int = syscall.O_CREAT // create a new file if none exists.
    //与创建文件一同使用,文件必须存在
    O_EXCL int = syscall.O_EXCL // used with O_CREATE, file must not exist
    //打开一个同步的文件流
    O_SYNC int = syscall.O_SYNC // open for synchronous I/O.
    //如果可能,打开时缩短文件
    O_TRUNC int = syscall.O_TRUNC // if possible, truncate file when opened.
)
```

## 打开模式

```go
//打开模式
const (
    ModeDir FileMode = 1 << (32 - 1 - iota) // d: is a directory 文件夹模式
    ModeAppend // a: append-only 追加模式
    ModeExclusive // l: exclusive use 单独使用
    ModeTemporary // T: temporary file (not backed up) 临时文件
    ModeSymlink // L: symbolic link 象征性的关联
    ModeDevice // D: device file 设备文件
    ModeNamedPipe // p: named pipe (FIFO) 命名管道
    ModeSocket // S: Unix domain socket Unix 主机 socket
    ModeSetuid // u: setuid 设置uid
    ModeSetgid // g: setgid 设置gid
    ModeCharDevice // c: Unix character device, when ModeDevice is set Unix 字符设备,当设备模式是设置Unix
    ModeSticky // t: sticky 黏滞位
    // Mask for the type bits. For regular files, none will be set. bit位遮盖.不变的文件设置为none
    ModeType = ModeDir | ModeSymlink | ModeNamedPipe | ModeSocket | ModeDevice
    ModePerm FileMode = 0777 // Unix permission bits 权限位.
)
```

## os.Create 创建文件

```go
f, err := os.Create(fileName)
defer f.Close()
```

```go
// file.go
func Create(name string) (*File, error) {
return OpenFile(name, O_RDWR|O_CREATE|O_TRUNC, 0666)
}
```

可以看到创建文件，`golang` 调用了 `OpenFile` 方法，并传入了 3 个 `flag` 以及指定了 0666 权限的打开方式。

```go
// OpenFile is the generalized open call; most users will use Open
// or Create instead. It opens the named file with specified flag
// (O_RDONLY etc.). If the file does not exist, and the O_CREATE flag
// is passed, it is created with mode perm (before umask). If successful,
// methods on the returned File can be used for I/O.
// If there is an error, it will be of type *PathError.
func OpenFile(name string, flag int, perm FileMode) (*File, error) {
    testlog.Open(name)
    f, err := openFileNolog(name, flag, perm)
    if err != nil {
        return nil, err
    }
    f.appendMode = flag&O_APPEND != 0
    
    return f, nil
}
```

注释翻译来说，就是 `OpenFile` 为最常规的打开调用方式。大部分开发者会用 `Open` 或者 `Create` 代替。如果文件不存在并且有 `O_Create` 标记，将会创建文件。

可以看到 `OpenFile` 核心就是调用了 `openFileNolog` 去打开一个文件，参数都是透传过去。

`windows` 和 `unix` 的代码并不一致，因为其背后操作系统以及文件系统都是不一样的，我们先来看看 `Unix`。

```go
// file_unix.go
// openFileNolog is the Unix implementation of OpenFile.
// Changes here should be reflected in openFdAt, if relevant.
func openFileNolog(name string, flag int, perm FileMode) (*File, error) {
    setSticky := false
    if !supportsCreateWithStickyBit && flag&O_CREATE != 0 && perm&ModeSticky != 0 {
        if _, err := Stat(name); IsNotExist(err) {
            setSticky = true
        }
    }
    
    var r int
    for {
        var e error
        r, e = syscall.Open(name, flag|syscall.O_CLOEXEC, syscallMode(perm))
        if e == nil {
            break
        }
    
        // We have to check EINTR here, per issues 11180 and 39237.
        if e == syscall.EINTR {
            continue
        }
    
        return nil, &PathError{Op: "open", Path: name, Err: e}
    }
    
    // open(2) itself won't handle the sticky bit on *BSD and Solaris
    if setSticky {
        setStickyBit(name)
    }
    
    // There's a race here with fork/exec, which we are
    // content to live with. See ../syscall/exec_unix.go.
    if !supportsCloseOnExec {
        syscall.CloseOnExec(r)
    }
    
    return newFile(uintptr(r), name, kindOpenFile), nil
}
```

先是通过运算符 & 去判断标识符是否存在，如果没有创建的标识符以及[黏滞位]()，则在通过 [`stat`](./stat.md) 文件不存在时报错。

可以看到 `unix` 在 `for` 里面循环去系统调用 `Open` 函数，如没有错误则退出循环。

`
调用 open 函数 O_CLOEXEC 模式打开的文件描述符在执行 exec 调用新程序中关闭，且为原子操作
`

当系统调用返回 `syscall.EINTR` 则继续执行，否则则返回对应错误。

> 为什么要用 for ？
> 可以从历史中看到，添加判断 `syscall.EINTR` 时用的还是 goto 语法，之前则没有任何循环。for 是为了防止系统调用返回 `syscall.EINTR` 时退出。 
> ![](https://files.catbox.moe/chzxyj.png)

后面逻辑为

* 当 `setSticky` 为 true 则 `Chmod` 文件添加 `ModeSticky`
* !supportsCloseOnExec 时系统调用 `CloseOnExec`

最终返回结果为 `newFile` 函数的结果，。

```go
// newFile is like NewFile, but if called from OpenFile or Pipe
// (as passed in the kind parameter) it tries to add the file to
// the runtime poller.
func newFile(fd uintptr, name string, kind newFileKind) *File {
    fdi := int(fd)
    if fdi < 0 {
        return nil
    }
    f := &File{&file{
        pfd: poll.FD{
            Sysfd:         fdi,
            IsStream:      true,
            ZeroReadIsEOF: true,
        },
        name:        name,
        stdoutOrErr: fdi == 1 || fdi == 2,
    }}
    
    pollable := kind == kindOpenFile || kind == kindPipe || kind == kindNonBlock
    
    // If the caller passed a non-blocking filedes (kindNonBlock),
    // we assume they know what they are doing so we allow it to be
    // used with kqueue.
    if kind == kindOpenFile {
        switch runtime.GOOS {
        case "darwin", "ios", "dragonfly", "freebsd", "netbsd", "openbsd":
            var st syscall.Stat_t
            err := ignoringEINTR(func() error {
                return syscall.Fstat(fdi, &st)
            })
            typ := st.Mode & syscall.S_IFMT
            // Don't try to use kqueue with regular files on *BSDs.
            // On FreeBSD a regular file is always
            // reported as ready for writing.
            // On Dragonfly, NetBSD and OpenBSD the fd is signaled
            // only once as ready (both read and write).
            // Issue 19093.
            // Also don't add directories to the netpoller.
            if err == nil && (typ == syscall.S_IFREG || typ == syscall.S_IFDIR) {
                pollable = false
            }
    
            // In addition to the behavior described above for regular files,
            // on Darwin, kqueue does not work properly with fifos:
            // closing the last writer does not cause a kqueue event
            // for any readers. See issue #24164.
            if (runtime.GOOS == "darwin" || runtime.GOOS == "ios") && typ == syscall.S_IFIFO {
                pollable = false
            }
        }
    }
    
    if err := f.pfd.Init("file", pollable); err != nil {
        // An error here indicates a failure to register
        // with the netpoll system. That can happen for
        // a file descriptor that is not supported by
        // epoll/kqueue; for example, disk files on
        // Linux systems. We assume that any real error
        // will show up in later I/O.
    } else if pollable {
        // We successfully registered with netpoll, so put
        // the file into nonblocking mode.
        if err := syscall.SetNonblock(fdi, true); err == nil {
            f.nonblock = true
        }
    }
    
    runtime.SetFinalizer(f.file, (*file).close)
    return f
}
```

`newFile` 先是判断系统调用 `Open` 返回的文件描述符的值是否小于0，后面则是构造 `File` 结构。

`kind` 传参是 `kindOpenFile`，`pollable` 为 `true`，用于后续系统调用 `SetNonblock` 为 `true`。

接下来我们来看看 `windows` 的代码。

```go
// file_windows.go
// openFileNolog is the Windows implementation of OpenFile.
func openFileNolog(name string, flag int, perm FileMode) (*File, error) {
    if name == "" {
        return nil, &PathError{Op: "open", Path: name, Err: syscall.ENOENT}
    }
    r, errf := openFile(name, flag, perm)
    if errf == nil {
        return r, nil
    }
    r, errd := openDir(name)
    if errd == nil {
        if flag&O_WRONLY != 0 || flag&O_RDWR != 0 {
            r.Close()
            return nil, &PathError{Op: "open", Path: name, Err: syscall.EISDIR}
        }
        return r, nil
    }
    return nil, &PathError{Op: "open", Path: name, Err: errf}
}
```

这里的代码也很简单，判断文件名，调用 `openFile`，没有错误则返回，否则调用 `openDir`。当没有错误以及有标识符 `O_WRONLY` 或 `O_RDWR` 时，关闭文件夹并且返回错误。

```go
// file_windows.go
func openFile(name string, flag int, perm FileMode) (file *File, err error) {
    r, e := syscall.Open(fixLongPath(name), flag|syscall.O_CLOEXEC, syscallMode(perm))
    if e != nil {
        return nil, e
    }
    return newFile(r, name, "file"), nil
}
```

`windows` 下的 `openFile` 显得简单多了，系统调用没有错误，则返回函数 `newFile`。

```go
// file_windows.go
// newFile returns a new File with the given file handle and name.
// Unlike NewFile, it does not check that h is syscall.InvalidHandle.
func newFile(h syscall.Handle, name string, kind string) *File {
    if kind == "file" {
        var m uint32
        if syscall.GetConsoleMode(h, &m) == nil {
            kind = "console"
        }
        if t, err := syscall.GetFileType(h); err == nil && t == syscall.FILE_TYPE_PIPE {
            kind = "pipe"
        }
    }
    
    f := &File{&file{
        pfd: poll.FD{
            Sysfd:         h,
            IsStream:      true,
            ZeroReadIsEOF: true,
        },
        name: name,
    }}
    runtime.SetFinalizer(f.file, (*file).close)
    
    // Ignore initialization errors.
    // Assume any problems will show up in later I/O.
    f.pfd.Init(kind, false)
    
    return f
}
```

可以看到除了构造 `file` 结构外，还调用了文件描述符的 `Init` 方法，把上面的 `kind` 传了进去。

> 文件描述符不在本节内容，以后会新开篇章详细讲讲

接下来我们看看报错后执行的 `openDir` 又做了什么。

```go
// file_windows.go
func openDir(name string) (file *File, err error) {
    var mask string
    
    path := fixLongPath(name)
    
    if len(path) == 2 && path[1] == ':' { // it is a drive letter, like C:
        mask = path + `*`
    } else if len(path) > 0 {
        lc := path[len(path)-1]
        if lc == '/' || lc == '\\' {
            mask = path + `*`
        } else {
            mask = path + `\*`
        }
    } else {
        mask = `\*`
    }
    maskp, e := syscall.UTF16PtrFromString(mask)
    if e != nil {
        return nil, e
    }
    d := new(dirInfo)
    r, e := syscall.FindFirstFile(maskp, &d.data)
    if e != nil {
        // FindFirstFile returns ERROR_FILE_NOT_FOUND when
        // no matching files can be found. Then, if directory
        // exists, we should proceed.
        if e != syscall.ERROR_FILE_NOT_FOUND {
            return nil, e
        }
        var fa syscall.Win32FileAttributeData
        pathp, e := syscall.UTF16PtrFromString(path)
        if e != nil {
            return nil, e
        }
        e = syscall.GetFileAttributesEx(pathp, syscall.GetFileExInfoStandard, (*byte)(unsafe.Pointer(&fa)))
        if e != nil {
            return nil, e
        }
        if fa.FileAttributes&syscall.FILE_ATTRIBUTE_DIRECTORY == 0 {
            return nil, e
        }
        d.isempty = true
    }
    d.path = path
    if !isAbs(d.path) {
        d.path, e = syscall.FullPath(d.path)
        if e != nil {
            return nil, e
        }
    }
    f := newFile(r, name, "dir")
    f.dirinfo = d
    return f, nil
}
```

先是判断参数 `name` 的格式，是否磁盘，例如`C:`，是否某些特定符号结尾等，生成参数 `mask`。

因为 `windows` 系统是使用 UTF-16 编码，所以需要把文件路径的字符串转成 UTF-16。

调用系统函数 `FindFirstFile` 并写入刚 `new` 的 `dirInfo`。

后面就是系统调用返回了详细路径，中间部分不作详细讲解。

到此为止，`os.Create` 终于讲完了。 

## os.OpenFile 写入文件

```go
f, err := os.OpenFile(fileName, os.O_WRONLY|os.O_TRUNC, 0600)
defer f.Close()
if err == nil {
	f.Write([]byte("text"))
}
```

```go
// file.go
// Write writes len(b) bytes from b to the File.
// It returns the number of bytes written and an error, if any.
// Write returns a non-nil error when n != len(b).
func (f *File) Write(b []byte) (n int, err error) {
    if err := f.checkValid("write"); err != nil {
        return 0, err
    }
    n, e := f.write(b)
    if n < 0 {
        n = 0
    }
    if n != len(b) {
        err = io.ErrShortWrite
    }
    
    epipecheck(f, e)
    
    if e != nil {
        err = f.wrapErr("write", e)
    }
    
    return n, err
}

// checkValid checks whether f is valid for use.
// If not, it returns an appropriate error, perhaps incorporating the operation name op.
func (f *File) checkValid(op string) error {
    if f == nil {
        return ErrInvalid
    }
    return nil
}
```

`Write` 函数首先会检查 `f` 是否为空，否则将调用 `write` 函数。

```go
// file_posix.go
// write writes len(b) bytes to the File.
// It returns the number of bytes written and an error, if any.
func (f *File) write(b []byte) (n int, err error) {
    n, err = f.pfd.Write(b)
    runtime.KeepAlive(f)
    return n, err
}
```

`write` 函数会调用操作系统对应的文件描述符的 `Write` 函数，关于 fd 的内容我之后将另开篇章讲解。

当写入成功后，会判断写入文件长度与参数长度是否一致，不一致则设置 `err` 为 `io.ErrShortWrite`。

## 小结

本节内容讲解的文件的创建、打开以及写入，但这只是比较浅层的 `golang` 包，实际与操作系统接触的内容并不多，如需要更加深入了解，可以看 [internal/poll/fd]() 的内容。