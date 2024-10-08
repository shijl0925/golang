# os — 平台无关的操作系统功能实现 #

`os` 包提供了平台无关的操作系统功能接口。尽管错误处理是 go 风格的，但设计是 Unix 风格的；所以，失败的调用会返回 `error` 而非错误码。通常 `error` 里会包含更多信息。例如，如果使用一个文件名的调用（如 Open、Stat）失败了，打印错误时会包含该文件名，错误类型将为 `*PathError`，其内部可以解包获得更多信息。

os 包规定为所有操作系统实现的接口都是一致的。有一些某个系统特定的功能，需要使用 `syscall` 获取。实际上，`os` 依赖于 `syscall`。在实际编程中，我们应该总是优先使用 `os` 中提供的功能，而不是 `syscall`。

下面是一个简单的例子，打开一个文件并从中读取一些数据：

```go
file, err := os.Open("file.go") // For read access.
if err != nil {
	log.Fatal(err)
}
```
如果打开失败，错误字符串是自解释的，例如：

`open file.go: no such file or directory`

而不像 C 语言，需要额外的函数（或宏）来解释错误码。

## 文件 I/O

在第一章，我们较全面的介绍了 Go 中的 I/O。本节，我们着重介绍文件相关的 I/O。因为 I/O 操作涉及到系统调用，在讲解时会涉及到 Unix   在这方面的系统调用。

在 Unix 系统调用中，所有 I/O 操作以文件描述符 ( 一个非负整数 , 通常是小整数 ) 来指代打开的文件。文件描述符用以表示所有类型的已打开文件，包括管道（pipe）、FIFO、socket、终端、设备和普通文件。这里，我们主要介绍普通文件的 I/O。

在 Go 中，文件描述符封装在 `os.File` 结构中，通过 `File.Fd()` 可以获得底层的文件描述符：fd。

按照惯例，大多数程序都期望能够使用 3 种标准的文件描述符：0- 标准输入；1- 标准输出；2- 标准错误。`os` 包提供了 3 个 `File` 对象，分别代表这 3 种标准描述符：`Stdin`、`Stdout` 和 `Stderr`，它们对应的文件名分别是：/dev/stdin、/dev/stdout 和 /dev/stderr。注意，这里说的文件名，并不一定存在，比如 Windows 下就没有。

### 打开一个文件：OpenFile

`OpenFile` 既能打开一个已经存在的文件，也能创建并打开一个新文件。

`func OpenFile(name string, flag int, perm FileMode) (*File, error)`

`OpenFile` 是一个更一般性的文件打开函数，大多数调用者都应用 `Open` 或 `Create` 代替本函数。它会使用指定的选项（如 O_RDONLY 等）、指定的模式（如 0666 等）打开指定名称的文件。如果操作成功，返回的文件对象可用于 I/O。如果出错，错误底层类型是 `*PathError`。

要打开的文件由参数 `name` 指定，它可以是绝对路径或相对路径（相对于进程当前工作目录），也可以是一个符号链接（会对其进行解引用）。

位掩码参数 `flag` 用于指定文件的访问模式，可用的值在 `os` 中定义为常量（以下值并非所有操作系统都可用）：

```
const (
    O_RDONLY int = syscall.O_RDONLY // 只读模式打开文件
    O_WRONLY int = syscall.O_WRONLY // 只写模式打开文件
    O_RDWR   int = syscall.O_RDWR   // 读写模式打开文件
    O_APPEND int = syscall.O_APPEND // 写操作时将数据附加到文件尾部
    O_CREATE int = syscall.O_CREAT  // 如果不存在将创建一个新文件
    O_EXCL   int = syscall.O_EXCL   // 和 O_CREATE 配合使用，文件必须不存在
    O_SYNC   int = syscall.O_SYNC   // 打开文件用于同步 I/O
    O_TRUNC  int = syscall.O_TRUNC  // 如果可能，打开时清空文件
)
```
其中，`O_RDONLY`、`O_WRONLY`、`O_RDWR` 应该只指定一个，剩下的通过 `|` 操作符来指定。该函数内部会给 `flags` 加上 `syscall.O_CLOEXEC`，在 fork 子进程时会关闭通过 `OpenFile` 打开的文件，即子进程不会重用该文件描述符。

*注意：由于历史原因，`O_RDONLY | O_WRONLY` 并非等于 `O_RDWR`，它们的值一般是 0、1、2。*

位掩码参数 `perm` 指定了文件的模式和权限位，类型是 `os.FileMode`，文件模式位常量定义在 `os` 中：

```go
const (
    // 单字符是被 String 方法用于格式化的属性缩写。
    ModeDir        FileMode = 1 << (32 - 1 - iota) // d: 目录
    ModeAppend                                     // a: 只能写入，且只能写入到末尾
    ModeExclusive                                  // l: 用于执行
    ModeTemporary                                  // T: 临时文件（非备份文件）
    ModeSymlink                                    // L: 符号链接（不是快捷方式文件）
    ModeDevice                                     // D: 设备
    ModeNamedPipe                                  // p: 命名管道（FIFO）
    ModeSocket                                     // S: Unix 域 socket
    ModeSetuid                                     // u: 表示文件具有其创建者用户 id 权限
    ModeSetgid                                     // g: 表示文件具有其创建者组 id 的权限
    ModeCharDevice                                 // c: 字符设备，需已设置 ModeDevice
    ModeSticky                                     // t: 只有 root/ 创建者能删除 / 移动文件
    
    // 覆盖所有类型位（用于通过 & 获取类型位），对普通文件，所有这些位都不应被设置
    ModeType = ModeDir | ModeSymlink | ModeNamedPipe | ModeSocket | ModeDevice
    ModePerm FileMode = 0777 // 覆盖所有 Unix 权限位（用于通过 & 获取类型位）
)
```
以上常量在所有操作系统都有相同的含义（可用时），因此文件的信息可以在不同的操作系统之间安全的移植。不是所有的位都能用于所有的系统，唯一共有的是用于表示目录的 `ModeDir` 位。

以上这些被定义的位是 `FileMode` 最重要的位。另外 9 个位（权限位）为标准 Unix rwxrwxrwx 权限（所有人都可读、写、运行）。

`FileMode` 还定义了几个方法，用于判断文件类型的 `IsDir()` 和 `IsRegular()`，用于获取权限的 `Perm()`。

返回的 `error`，具体实现是 `*os.PathError`，它会记录具体操作、文件路径和错误原因。

另外，在 `OpenFile` 内部会调用 `NewFile`，来得到 `File` 对象。

**使用方法**

打开一个文件，一般通过 `Open` 或 `Create`，我们看这两个函数的实现。

```go
func Open(name string) (*File, error) {
	return OpenFile(name, O_RDONLY, 0)
}

func Create(name string) (*File, error) {
	return OpenFile(name, O_RDWR|O_CREATE|O_TRUNC, 0666)
}
```

### 读取文件内容：Read

`func (f *File) Read(b []byte) (n int, err error)`

`Read` 方法从 `f` 中读取最多 `len(b)` 字节数据并写入 `b`。它返回读取的字节数和可能遇到的任何错误。文件终止标志是读取 0 个字节且返回值 err 为 `io.EOF`。

从方法声明可以知道，`File` 实现了 `io.Reader` 接口。

`Read` 对应的系统调用是 `read`。

对比下 `ReadAt` 方法：

`func (f *File) ReadAt(b []byte, off int64) (n int, err error)`

`ReadAt` 从指定的位置（相对于文件开始位置）读取长度为 `len(b)` 个字节数据并写入 `b`。它返回读取的字节数和可能遇到的任何错误。当 n<len(b) 时，本方法总是会返回错误；如果是因为到达文件结尾，返回值 err 会是 `io.EOF`。它对应的系统调用是 `pread`。

**`Read` 和 `ReadAt` 的区别**：前者从文件当前偏移量处读，且会改变文件当前的偏移量；而后者从 `off` 指定的位置开始读，且**不会改变**文件当前偏移量。


### 读取文件内容的三种方法
* 使用`Read` 按段读取
    ```go
    package main

    import (
        "fmt"
        "io"
        "log"
        "os"
    )

    func main() {
        file, err := os.OpenFile("vc.json", os.O_RDONLY, 0)
        if err != nil {
            log.Fatalf("Failed to open file: %v", err)
        }
        defer file.Close()

        for {
            buf := make([]byte, 1024)
            n, err := file.Read(buf)
            if err != nil {
                if err == io.EOF {
                    break
                }
                log.Fatalf("Failed to read file: %v", err)
            }
            if n == 0 {
                break
            }
            fmt.Println(string(buf[:n]))
        }
    }
    ```

* 使用 `bufio.NewReader` 和 `Reader.ReadString` ，减少磁盘的操作。
    ```go
    package main

    import (
        "bufio"
        "fmt"
        "io"
        "log"
        "os"
        "strings"
    )

    func main() {
        file, err := os.OpenFile("vc.json", os.O_RDONLY, 0)
        if err != nil {
            log.Fatalf("Failed to open file: %v", err)
        }
        defer file.Close()

        reader := bufio.NewReader(file)
        for {
            line, err := reader.ReadString('\n')
            if err != nil {
                if err == io.EOF {
                    break
                }
                log.Fatalf("Error reading line: %v", err)
            }
            fmt.Println(strings.TrimRight(line, "\n"))
        }
    }
    ```
* 使用`os.ReadFile`读取整个文件的所有内容

    ```go
    package main

    import (
        "fmt"
        "log"
        "os"
    )

    func main() {
        data, err := os.ReadFile("vc.json")
        if err != nil {
            log.Fatalf("Failed to read file: %v", err)
        }
        fmt.Println(string(data))
    }
    ```

### 数据写入文件：Write

`func (f *File) Write(b []byte) (n int, err error)`

`Write` 向文件中写入 `len(b)` 字节数据。它返回写入的字节数和可能遇到的任何错误。如果返回值 `n!=len(b)`，本方法会返回一个非 nil 的错误。

从方法声明可以知道，`File` 实现了 `io.Writer` 接口。

`Write` 对应的系统调用是 `write`。

`Write` 与 `WriteAt` 的区别同 `Read` 与 `ReadAt` 的区别一样。为了方便，还提供了 `WriteString` 方法，它实际是对 `Write` 的封装。

注意：`Write` 调用成功并不能保证数据已经写入磁盘，因为内核会缓存磁盘的 I/O 操作。如果希望立刻将数据写入磁盘（一般场景不建议这么做，因为会影响性能），有两种办法：

    1. 打开文件时指定 `os.O_SYNC`；
    2. 调用 `File.Sync()` 方法。

说明：`File.Sync()` 底层调用的是 `fsync` 系统调用，这会将数据和元数据都刷到磁盘；如果只想刷数据到磁盘（比如，文件大小没变，只是变了文件数据），需要自己封装，调用 `fdatasync` 系统调用。（`syscall.Fdatasync`）

### 文件写入的三种方法
* `Write`和`WriteString`写入操作
    ```go
    package main

    import (
        "log"
        "os"
    )

    func main() {
        file, err := os.OpenFile("test.txt", os.O_CREATE|os.O_WRONLY, 0644)
        if err != nil {
            log.Fatalf("Failed to open file: %v", err)
        }
        defer file.Close()

        if _, err := file.Write([]byte("Hello, World!")); err != nil {
            log.Fatalf("Failed to write to file: %v", err)
        }

        //if _, err := file.WriteString("Hello, World!"); err != nil {
        //	log.Fatalf("Failed to write to file: %v", err)
        //}
    }
    ```

* `bufio.NewWriter` 先往缓冲区写,然后才写到磁盘中去。
    ```go
    package main

    import (
        "bufio"
        "log"
        "os"
    )

    func main() {
        file, err := os.OpenFile("test.txt", os.O_CREATE|os.O_WRONLY, 0644)
        if err != nil {
            log.Fatalf("Failed to open file: %v", err)
        }
        defer file.Close()

        writer := bufio.NewWriter(file)
        defer writer.Flush()

        if _, err := writer.WriteString("Hello, World!\n"); err != nil {
            log.Fatalf("Failed to write to file: %v", err)
        }
    }
    ```

* `os.WriteFile` 写入操作
    ```go
    package main

    import (
        "log"
        "os"
    )

    func main() {
        err := os.WriteFile("test.txt", []byte("hello world"), 0644)
        if err != nil {
            log.Fatalf("Failed to write to file: %v", err)
        }
    }
    ```

### 关闭文件：Close

`close()` 系统调用关闭一个打开的文件描述符，并将其释放回调用进程，供该进程继续使用。当进程终止时，将自动关闭其已打开的所有文件描述符。

`func (f *File) Close() error`

`os.File.Close()` 是对 `close()` 的封装。我们应该养成关闭不需要的文件的良好编程习惯。文件描述符是资源，Go 的 gc 是针对内存的，并不会自动回收资源，如果不关闭文件描述符，长期运行的服务可能会把文件描述符耗尽。

所以，通常的写法如下：

```go
file, err := os.Open("/tmp/studygolang.txt")
if err != nil {
	// 错误处理，一般会阻止程序往下执行
	return
}
defer file.Close()
```

**关于返回值 `error`**

以下两种情况会导致 `Close` 返回错误：

    1. 关闭一个未打开的文件；
    2. 两次关闭同一个文件；

通常，我们不会去检查 `Close` 的错误。

### 改变文件偏移量：Seek

对于每个打开的文件，系统内核会记录其文件偏移量，有时也将文件偏移量称为读写偏移量或指针。文件偏移量是指执行下一个 `Read` 或 `Write` 操作的文件其实位置，会以相对于文件头部起始点的文件当前位置来表示。文件第一个字节的偏移量为 0。

文件打开时，会将文件偏移量设置为指向文件开始，以后每次 `Read` 或 `Write` 调用将自动对其进行调整，以指向已读或已写数据后的下一个字节。因此，连续的 `Read` 和 `Write` 调用将按顺序递进，对文件进行操作。

而 `Seek` 可以调整文件偏移量。方法定义如下：

`func (f *File) Seek(offset int64, whence int) (ret int64, err error)`

`Seek` 设置下一次读 / 写的位置。offset 为相对偏移量，而 whence 决定相对位置：0 为相对文件开头，1 为相对当前位置，2 为相对文件结尾。它返回新的偏移量（相对开头）和可能的错误。使用中，whence 应该使用 `os` 包中的常量：`SEEK_SET`、`SEEK_CUR` 和 `SEEK_END`。

注意：`Seek` 只是调整内核中与文件描述符相关的文件偏移量记录，并没有引起对任何物理设备的访问。

一些 `Seek` 的使用例子（file 为打开的文件对象），注释说明了将文件偏移量移动到的具体位置：

```go
file.Seek(0, os.SEEK_SET)	// 文件开始处
file.Seek(0, SEEK_END)		// 文件结尾处的下一个字节
file.Seek(-1, SEEK_END)		// 文件最后一个字节
file.Seek(-10, SEEK_CUR) 	// 当前位置前 10 个字节
file.Seek(1000, SEEK_END)	// 文件结尾处的下 1001 个字节
```
最后一个例子在文件中会产生“空洞”。

`Seek` 对应系统调用 `lseek`。该系统调用并不适用于所有类型，不允许将 `lseek ` 应用于管道、FIFO、socket 或 终端。

## 截断文件

`trucate` 和 `ftruncate` 系统调用将文件大小设置为 `size` 参数指定的值；Go 语言中相应的包装函数是 `os.Truncate` 和 `os.File.Truncate`。

```go
func Truncate(name string, size int64) error
func (f *File) Truncate(size int64) error
```
如果文件当前长度大于参数 `size`，调用将丢弃超出部分，若小于参数 `size`，调用将在文件尾部添加一系列空字节或是一个文件空洞。

它们之间的区别在于如何指定操作文件：

    1. `Truncate` 以路径名称字符串来指定文件，并要求可访问该文件（即对组成路径名的各目录拥有可执行 (x) 权限），且对文件拥有写权限。若文件名为符号链接，那么调用将对其进行解引用。
    2. 很明显，调用 `File.Truncate` 前，需要先以可写方式打开操作文件，该方法不会修改文件偏移量。

## 文件属性

文件属性，也即文件元数据。在 Go 中，文件属性具体信息通过 `os.FileInfo` 接口获取。函数 `Stat`、`Lstat` 和 `File.Stat` 可以得到该接口的实例。这三个函数对应三个系统调用：`stat`、`lstat` 和 `fstat`。

这三个函数的区别：

1. `stat` 会返回所命名文件的相关信息。
2. `lstat` 与 `stat` 类似，区别在于如果文件是符号链接，那么所返回的信息针对的是符号链接自身（而非符号链接所指向的文件）。
3. `fstat` 则会返回由某个打开文件描述符（Go 中则是当前打开文件 File）所指代文件的相关信息。

`Stat` 和 `Lstat` 无需对其所操作的文件本身拥有任何权限，但针对指定 name 的父目录要有执行（搜索）权限。而只要 `File` 对象 ok，`File.Stat` 总是成功。

`FileInfo` 接口如下：

```go
type FileInfo interface {
    Name() string       // 文件的名字（不含扩展名）
    Size() int64        // 普通文件返回值表示其大小；其他文件的返回值含义各系统不同
    Mode() FileMode     // 文件的模式位
    ModTime() time.Time // 文件的修改时间
    IsDir() bool        // 等价于 Mode().IsDir()
    Sys() interface{}   // 底层数据来源（可以返回 nil）
}
```

`Sys()` 底层数据的 C 语言 结构 `statbuf` 格式如下：

```go
struct stat {
	dev_t	st_dev;	// 设备 ID
	ino_t	st_ino;	// 文件 i 节点号
	mode_t	st_mode;	// 位掩码，文件类型和文件权限
	nlink_t	st_nlink;	// 硬链接数
	uid_t	st_uid;	// 文件属主，用户 ID
	gid_t	st_gid;	// 文件属组，组 ID
	dev_t	st_rdev;	// 如果针对设备 i 节点，则此字段包含主、辅 ID
	off_t	st_size;	// 常规文件，则是文件字节数；符号链接，则是链接所指路径名的长度，字节为单位；对于共享内存对象，则是对象大小
	blksize_t	st_blsize;	// 分配给文件的总块数，块大小为 512 字节
	blkcnt_t	st_blocks;	// 实际分配给文件的磁盘块数量
	time_t	st_atime;		// 对文件上次访问时间
	time_t	st_mtime;		// 对文件上次修改时间
	time_t	st_ctime;		// 文件状态发生改变的上次时间
}
```
Go 中 `syscal.Stat_t` 与该结构对应。

如果我们要获取 `FileInfo` 接口没法直接返回的信息，比如想获取文件的上次访问时间，示例如下：

```go
fileInfo, err := os.Stat("test.log")
if err != nil {
	log.Fatal(err)
}
sys := fileInfo.Sys()
stat := sys.(*syscall.Stat_t)
fmt.Println(time.Unix(stat.Atimespec.Unix()))
```

### 改变文件时间戳

可以显式改变文件的访问时间和修改时间。

`func Chtimes(name string, atime time.Time, mtime time.Time) error`

`Chtimes` 修改 name 指定的文件对象的访问时间和修改时间，类似 Unix 的 utime() 或 utimes() 函数。底层的文件系统可能会截断 / 舍入时间单位到更低的精确度。如果出错，会返回 `*PathError` 类型的错误。在 Unix 中，底层实现会调用 `utimenstat()`，它提供纳秒级别的精度。

### 文件属主

每个文件都有一个与之关联的用户 ID（UID）和组 ID（GID），籍此可以判定文件的属主和属组。系统调用 `chown`、`lchown` 和 `fchown` 可用来改变文件的属主和属组，Go 中对应的函数或方法：

```go
func Chown(name string, uid, gid int) error
func Lchown(name string, uid, gid int) error
func (f *File) Chown(uid, gid int) error
```
它们的区别和上文提到的 `Stat` 相关函数类似。

### 文件权限

这里介绍是应用于文件和目录的权限方案，尽管此处讨论的权限主要是针对普通文件和目录，但其规则可适用于所有文件类型，包括设备文件、FIFO 以及 Unix 域套接字等。

#### 普通文件的权限

如前所述，`os.FileMode` 或 C 结构 stat 中的 `st_mod` 的低 12 位定义了文件权限。其中前 3 位为专用位，分别是 set-user-ID 位、set-group-ID 位和 sticky 位。其余 9 位则构成了定义权限的掩码，分别授予访问文件的各类用户。文件权限掩码分为 3 类：

- Owner（亦称为 user）：授予文件属主的权限。
- Group：授予文件属组成员用户的权限。
- Other：授予其他用户的权限。

可为每一类用户授予的权限如下：

- Read：可阅读文件的内容。
- Write：可更改文件的内容。
- Execute：可以执行文件（如程序或脚本）。

Unix 中表示：rwxrwxrwx。

#### 目录权限

目录与文件拥有相同的权限方案，只是对 3 种权限的含义另有所指。

- 读权限：可列出（比如，通过 ls 命令）目录之下的内容（即目录下的文件名）。
- 写权限：可在目录内创建、删除文件。注意，要删除文件，对文件本身无需有任何权限。
- 可执行权限：可访问目录中的文件。因此，有时也将对目录的执行权限称为 search（搜索）权限。

访问文件时，需要拥有对路径名所列所有目录的执行权限。例如，想读取文件 `/home/studygolang/abc`，则需拥有对目录 `/`、`/home` 以及 `/home/studygolang` 的执行权限（还要有对文件 `abc` 自身的读权限）。

#### 相关函数或方法

在文件相关操作报错时，可以通过 `os.IsPermission` 检查是否是权限的问题。

`func IsPermission(err error) bool`

返回一个布尔值说明该错误是否表示因权限不足要求被拒绝。ErrPermission 和一些系统调用错误会使它返回真。

另外，`syscall.Access` 可以获取文件的权限。这对应系统调用 `access`。

#### Sticky 位

除了 9 位用来表明属主、属组和其他用户的权限外，文件权限掩码还另设有 3 个附加位，分别是 set-user-ID(bit 04000)、set-group-ID(bit 02000) 和 sticky(bit 01000) 位。set-user-ID 和 set-group-ID 权限位将在进程章节介绍。这里介绍 sticky 位。

Sticky 位一般用于目录，起限制删除位的作用，表明仅当非特权进程具有对目录的写权限，且为文件或目录的属主时，才能对目录下的文件进行删除和重命名操作。根据这个机制来创建为多个用户共享的一个目录，各个用户可在其下创建或删除属于自己的文件，但不能删除隶属于其他用户的文件。`/tmp` 目录就设置了 sticky 位，正是出于这个原因。

`chmod` 命令或系统调用可以设置文件的 sticky 位。若对某文件设置了 sticky 位，则 `ls -l` 显示文件时，会在其他用户执行权限字段上看到字母 t（有执行权限时） 或 T（无执行权限时）。

`os.Chmod` 和 `os.File.Chmod` 可以修改文件权限（包括 sticky 位），分别对应系统调用 `chmod` 和 `fchmod`。

```go
func main() {
    file, err := os.Create("studygolang.txt")
    if err != nil {
        log.Fatal("error:", err)
    }
    defer file.Close()

    fileMode := getFileMode(file)
    log.Println("file mode:", fileMode)
    file.Chmod(fileMode | os.ModeSticky)

    log.Println("change after, file mode:", getFileMode(file))
}

func getFileMode(file *os.File) os.FileMode {
    fileInfo, err := file.Stat()
    if err != nil {
        log.Fatal("file stat error:", err)
    }

    return fileInfo.Mode()
}

// Output:
// 2016/06/18 15:59:06 file mode: -rw-rw-r--
// 2016/06/18 15:59:06 change after, file mode: trw-rw-r--
// ls -l 看到的 studygolang.tx 是：-rw-rw-r-T
// 当然这里是给文件设置了 sticky 位，对权限不起作用。系统会忽略它。
```

## 目录与链接

在 Unix 文件系统中，目录的存储方式类似于普通文件。目录和普通文件的区别有二：

- 在其 i-node 条目中，会将目录标记为一种不同的文件类型。
- 目录是经特殊组织而成的文件。本质上说就是一个表格，包含文件名和 i-node 标号。

### 创建和移除（硬）链接

硬链接是针对文件而言的，目录不允许创建硬链接。

`link` 和 `unlink` 系统调用用于创建和移除（硬）链接。Go 中 `os.Link` 对应 `link` 系统调用；但 `os.Remove` 的实现会先执行 `unlink` 系统调用，如果要移除的是目录，则 `unlink` 会失败，这时 `Remove` 会再调用 `rmdir` 系统调用。

`func Link(oldname, newname string) error`

`Link` 创建一个名为 newname 指向 oldname 的硬链接。如果出错，会返回 `*LinkError` 类型的错误。

`func Remove(name string) error`

`Remove` 删除 name 指定的文件或目录。如果出错，会返回 `*PathError` 类型的错误。如果目录不为空，`Remove` 会返回失败。

### 更改文件名

系统调用 `rename` 既可以重命名文件，又可以将文件移至同一个文件系统中的另一个目录。该系统调用既可以用于文件，也可以用于目录。相关细节，请查阅相关资料。

Go 中的 `os.Rename` 是对应的封装函数。

`func Rename(oldpath, newpath string) error`

`Rename` 修改一个文件的名字或移动一个文件。如果 `newpath` 已经存在，则替换它。注意，可能会有一些个操作系统特定的限制。

### 使用符号链接

`symlink` 系统调用用于为指定路径名创建一个新的符号链接（想要移除符号链接，使用 `unlink`）。Go 中的 `os.Symlink` 是对应的封装函数。

`func Symlink(oldname, newname string) error`

`Symlink` 创建一个名为 `newname` 指向 `oldname` 的符号链接。如果出错，会返回 `*LinkError` 类型的错误。

由 `oldname` 所命名的文件或目录在调用时无需存在。因为即便当时存在，也无法阻止后来将其删除。这时，`newname` 成为“悬空链接”，其他系统调用试图对其进行解引用操作都将错误（通常错误号是 ENOENT）。

有时候，我们希望通过符号链接，能获取其所指向的路径名。系统调用 `readlink` 能做到，Go 的封装函数是 `os.Readlink`：

`func Readlink(name string) (string, error)`

`Readlink` 获取 `name` 指定的符号链接指向的文件的路径。如果出错，会返回 `*PathError` 类型的错误。我们看看 `Readlink` 的实现。

```go
func Readlink(name string) (string, error) {
	for len := 128; ; len *= 2 {
		b := make([]byte, len)
		n, e := fixCount(syscall.Readlink(name, b))
		if e != nil {
			return "", &PathError{"readlink", name, e}
		}
		if n < len {
			return string(b[0:n]), nil
		}
	}
}
```
这里之所以用循环，是因为我们没法知道文件的路径到底多长，如果 `b` 长度不够，文件名会被截断，而 `readlink` 系统调用无非分辨所返回的字符串到底是经过截断处理，还是恰巧将 `b` 填满。这里采用的验证方法是分配一个更大的（两倍）`b` 并再次调用 `readlink`。

### 创建和移除目录

`mkdir` 系统调用创建一个新目录，Go 中的 `os.Mkdir` 是对应的封装函数。

`func Mkdir(name string, perm FileMode) error`

`Mkdir` 使用指定的权限和名称创建一个目录。如果出错，会返回 `*PathError` 类型的错误。

`name` 参数指定了新目录的路径名，可以是相对路径，也可以是绝对路径。如果已经存在，则调用失败并返回 `os.ErrExist` 错误。

`perm` 参数指定了新目录的权限。对该位掩码值的指定方式和 `os.OpenFile` 相同，也可以直接赋予八进制数值。注意，`perm` 值还将于进程掩码相与（&）。如果 `perm` 中设置了 sticky 位，那么将对新目录设置该权限。

因为 `Mkdir` 所创建的只是路径名中的最后一部分，如果父目录不存在，创建会失败。`os.MkdirAll` 用于递归创建所有不存在的目录。

建议读者阅读下 `os.MkdirAll` 的源码，了解其实现方式、技巧。

`rmdir` 系统调用移除一个指定的目录，目录可以是绝对路径或相对路径。在讲解 `unlink` 时，已经介绍了 Go 中的 `os.Remove`。注意，这里要求目录必须为空。为了方便使用，Go 中封装了一个 `os.RemoveAll` 函数：

`func RemoveAll(path string) error`

`RemoveAll` 删除 `path` 指定的文件，或目录及它包含的任何下级对象。它会尝试删除所有东西，除非遇到错误并返回。如果 `path` 指定的对象不存在，`RemoveAll` 会返回 nil 而不返回错误。

`RemoveAll` 的内部实现逻辑如下：

1. 调用 `Remove` 尝试进行删除，如果成功或返回 `path` 不存在，则直接返回 nil；
2. 调用 `Lstat` 获取 `path` 信息，以便判断是否是目录。注意，这里使用 `Lstat`，表示不对符号链接解引用；
3. 调用 `Open` 打开目录，递归读取目录中内容，执行删除操作。

阅读 `RemoveAll` 源码，可以掌握马上要介绍的读目录内容或遍历目录。

### 读目录

`POSIX` 与 `SUS` 定义了读取目录相关的 C 语言标准，各个操作系统提供的系统调用却不尽相同。Go 没有基于 C 语言，而是自己通过系统调用实现了读目录功能。

`func (f *File) Readdirnames(n int) (names []string, err error)`

`Readdirnames` 读取目录 `f` 的内容，返回一个最多有 `n` 个成员的[]string，切片成员为目录中文件对象的名字，采用目录顺序。对本函数的下一次调用会返回上一次调用未读取的内容的信息。

如果 n>0，`Readdirnames` 函数会返回一个最多 n 个成员的切片。这时，如果 `Readdirnames` 返回一个空切片，它会返回一个非 nil 的错误说明原因。如果到达了目录 `f` 的结尾，返回值 err 会是 `io.EOF`。

如果 n<=0，`Readdirnames` 函数返回目录中剩余所有文件对象的名字构成的切片。此时，如果 `Readdirnames` 调用成功（读取所有内容直到结尾），它会返回该切片和 nil 的错误值。如果在到达结尾前遇到错误，会返回之前成功读取的名字构成的切片和该错误。

`func (f *File) Readdir(n int) (fi []FileInfo, err error)`

`Readdir` 内部会调用 `Readdirnames`，将得到的 `names` 构造路径，通过 `Lstat` 构造出 `[]FileInfo`。

列出某个目录的文件列表示例程序见 [dirtree](/code/src/chapter06/os/dirtree/main.go)。
