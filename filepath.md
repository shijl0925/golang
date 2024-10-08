# path/filepath — 兼容操作系统的文件路径操作 #

`path/filepath` 包涉及到路径操作时，路径分隔符使用 `os.PathSeparator`。不同系统，路径表示方式有所不同，比如 Unix 和 Windows 差别很大。本包能够处理所有的文件路径，不管是什么系统。

注意，路径操作函数并不会校验路径是否真实存在。

## 解析路径名字符串

`Dir()` 和 `Base()` 函数将一个路径名字符串分解成目录和文件名两部分。（注意一般情况，这些函数与 Unix 中 dirname 和 basename 命令类似，但如果路径以 `/` 结尾，`Dir` 的行为和 `dirname` 不太一致。）

```go
func Dir(path string) string
func Base(path string) string
```
`Dir` 返回路径中除去最后一个路径元素的部分，即该路径最后一个元素所在的目录。在使用 `Split` 去掉最后一个元素后，会简化路径并去掉末尾的斜杠。如果路径是空字符串，会返回 "."；如果路径由 1 到多个斜杠后跟 0 到多个非斜杠字符组成，会返回 "/"；其他任何情况下都不会返回以斜杠结尾的路径。

`Base` 函数返回路径的最后一个元素。在提取元素前会去掉末尾的斜杠。如果路径是 ""，会返回 "."；如果路径是只有一个斜杆构成的，会返回 "/"。

比如，给定路径名 `/home/polaris/studygolang.go`，`Dir` 返回 `/home/polaris`，而 `Base` 返回 `studygolang.go`。

如果给定路径名 `/home/polaris/studygolang/`，`Dir` 返回 `/home/polaris/studygolang`（这与 Unix 中的 dirname 不一致，dirname 会返回 /home/polaris），而 `Base` 返回 `studygolang`。

有人提出此问题，见[issue13199](https://github.com/golang/go/issues/13199)，不过官方认为这不是问题，如果需要和 `dirname` 一样的功能，应该自己处理，比如在调用 `Dir` 之前，先将末尾的 `/` 去掉。

此外，`Ext` 可以获得路径中文件名的扩展名。

`func Ext(path string) string`

`Ext` 函数返回 `path` 文件扩展名。扩展名是路径中最后一个从 `.` 开始的部分，包括 `.`。如果该元素没有 `.` 会返回空字符串。

## 相对路径和绝对路径

某个进程都会有当前工作目录（进程相关章节会详细介绍），一般的相对路径，就是针对进程当前工作目录而言的。当然，可以针对某个目录指定相对路径。

绝对路径，在 Unix 中，以 `/` 开始；在 Windows 下以某个盘符开始，比如 `C:\Program Files`。

`func IsAbs(path string) bool`

`IsAbs` 返回路径是否是一个绝对路径。而

`func Abs(path string) (string, error)`

`Abs` 函数返回 `path` 代表的绝对路径，如果 `path` 不是绝对路径，会加入当前工作目录以使之成为绝对路径。因为硬链接的存在，不能保证返回的绝对路径是唯一指向该地址的绝对路径。在 `os.Getwd` 出错时，`Abs` 会返回该错误，一般不会出错，如果路径名长度超过系统限制，则会报错。

`func Rel(basepath, targpath string) (string, error)`

`Rel` 函数返回一个相对路径，将 `basepath` 和该路径用路径分隔符连起来的新路径在词法上等价于 `targpath`。也就是说，`Join(basepath, Rel(basepath, targpath))` 等价于 `targpath`。如果成功执行，返回值总是相对于 `basepath` 的，即使 `basepath` 和 `targpath` 没有共享的路径元素。如果两个参数一个是相对路径而另一个是绝对路径，或者 `targpath` 无法表示为相对于 `basepath` 的路径，将返回错误。

```go
fmt.Println(filepath.Rel("/home/polaris/studygolang", "/home/polaris/studygolang/src/logic/topic.go"))
fmt.Println(filepath.Rel("/home/polaris/studygolang", "/data/studygolang"))

// Output:
// src/logic/topic.go <nil>
// ../../../data/studygolang <nil>
```

## 路径的切分和拼接

对于一个常规文件路径，我们可以通过 `Split` 函数得到它的目录路径和文件名：

`func Split(path string) (dir, file string)`

`Split` 函数根据最后一个路径分隔符将路径 `path` 分隔为目录和文件名两部分（`dir` 和 `file`）。如果路径中没有路径分隔符，函数返回值 `dir` 为空字符串，`file` 等于 `path`；反之，如果路径中最后一个字符是 `/`，则 `dir` 等于 `path`，`file` 为空字符串。返回值满足 `path == dir+file`。`dir` 非空时，最后一个字符总是 `/`。

```go
// dir == /home/polaris/，file == studygolang
filepath.Split("/home/polaris/studygolang")

// dir == /home/polaris/studygolang/，file == ""
filepath.Split("/home/polaris/studygolang/")

// dir == ""，file == studygolang
filepath.Split("studygolang")
```
相对路径到绝对路径的转变，需要经过路径的拼接。`Join` 用于将多个路径拼接起来，会根据情况添加路径分隔符。

`func Join(elem ...string) string`

`Join` 函数可以将任意数量的路径元素放入一个单一路径里，会根据需要添加路径分隔符。结果是经过 `Clean` 的，所有的空字符串元素会被忽略。对于拼接路径的需求，我们应该总是使用 `Join` 函数来处理。

有时，我们需要分割 `PATH` 或 `GOPATH` 之类的环境变量（这些路径被特定于 `OS` 的列表分隔符连接起来），`filepath.SplitList` 就是这个用途：

`func SplitList(path string) []string`

注意，与 `strings.Split` 函数的不同之处是：对 ""，SplitList 返回[]string{}，而 `strings.Split` 返回 []string{""}。`SplitList` 内部调用的是 `strings.Split`。

## 规整化路径

`func Clean(path string) string`

`Clean` 函数通过单纯的词法操作返回和 `path` 代表同一地址的最短路径。

它会不断的依次应用如下的规则，直到不能再进行任何处理：

1. 将连续的多个路径分隔符替换为单个路径分隔符
2. 剔除每一个 `.` 路径名元素（代表当前目录）
3. 剔除每一个路径内的 `..` 路径名元素（代表父目录）和它前面的非 `..` 路径名元素
4. 剔除开始于根路径的 `..` 路径名元素，即将路径开始处的 `/..` 替换为 `/`（假设路径分隔符是 `/`）

返回的路径只有其代表一个根地址时才以路径分隔符结尾，如 Unix 的 `/` 或 Windows 的 `C:\`。

如果处理的结果是空字符串，Clean 会返回 `.`，代表当前路径。

## 符号链接指向的路径名

在上一节 `os` 包中介绍了 `Readlink`，可以读取符号链接指向的路径名。不过，如果原路径中又包含符号链接，`Readlink` 却不会解析出来。`filepath.EvalSymlinks` 会将所有路径的符号链接都解析出来。除此之外，它返回的路径，是直接可访问的。

`func EvalSymlinks(path string) (string, error)`

如果 `path` 或返回值是相对路径，则是相对于进程当前工作目录。

`os.Readlink` 和 `filepath.EvalSymlinks` 区别示例程序：

```go
// 在当前目录下创建一个 studygolang.txt 文件和一个 symlink 目录，在 symlink 目录下对 studygolang.txt 建一个符号链接 studygolang.txt.2
fmt.Println(filepath.EvalSymlinks("symlink/studygolang.txt.2"))
fmt.Println(os.Readlink("symlink/studygolang.txt.2"))

// Ouput:
// studygolang.txt <nil>
// ../studygolang.txt <nil>
```

## 文件路径匹配

`func Match(pattern, name string) (matched bool, err error)`

`Match` 指示 `name` 是否和 shell 的文件模式匹配。模式语法如下：

```go
pattern:
	{ term }
term:
	'*'         匹配 0 或多个非路径分隔符的字符
	'?'         匹配 1 个非路径分隔符的字符
	'[' [ '^' ] { character-range } ']'  
				  字符组（必须非空）
	c           匹配字符 c（c != '*', '?', '\\', '['）
	'\\' c      匹配字符 c
character-range:
	c           匹配字符 c（c != '\\', '-', ']'）
	'\\' c      匹配字符 c
	lo '-' hi   匹配区间[lo, hi]内的字符
```
匹配要求 `pattern` 必须和 `name` 全匹配上，不只是子串。在 Windows 下转义字符被禁用。

`Match` 函数很少使用，搜索了一遍，标准库没有用到这个函数。而 `Glob` 函数在模板标准库中被用到了。

`func Glob(pattern string) (matches []string, err error)`

`Glob` 函数返回所有匹配了 模式字符串 `pattern` 的文件列表或者 nil（如果没有匹配的文件）。`pattern` 的语法和 `Match` 函数相同。`pattern` 可以描述多层的名字，如 `/usr/*/bin/ed`（假设路径分隔符是 `/`）。

注意，`Glob` 会忽略任何文件系统相关的错误，如读目录引发的 I/O 错误。唯一的错误和 `Match` 一样，在 `pattern` 不合法时，返回 `filepath.ErrBadPattern`。返回的结果是根据文件名字典顺序进行了排序的。

`Glob` 的常见用法，是读取某个目录下所有的文件，比如写单元测试时，读取 `testdata` 目录下所有测试数据：

`filepath.Glob("testdata/*.input")`

## 遍历目录

在介绍 `os` 时，讲解了读取目录的方法，并给出了一个遍历目录的示例。在 `filepath` 中，提供了 `Walk` 函数，用于遍历目录树。

`func Walk(root string, walkFn WalkFunc) error`

`Walk` 函数会遍历 `root` 指定的目录下的文件树，对每一个该文件树中的目录和文件都会调用 `walkFn`，包括 `root` 自身。所有访问文件 / 目录时遇到的错误都会传递给 `walkFn` 过滤。文件是按字典顺序遍历的，这让输出更漂亮，但也导致处理非常大的目录时效率会降低。`Walk` 函数不会遍历文件树中的符号链接（快捷方式）文件包含的路径。

`walkFn` 的类型 `WalkFunc` 的定义如下：

`type WalkFunc func(path string, info os.FileInfo, err error) error`

`Walk` 函数对每一个文件 / 目录都会调用 `WalkFunc` 函数类型值。调用时 `path` 参数会包含 `Walk` 的 `root` 参数作为前缀；就是说，如果 `Walk` 函数的 `root` 为 "dir"，该目录下有文件 "a"，将会使用 "dir/a" 作为调用 `walkFn` 的参数。`walkFn` 参数被调用时的 `info` 参数是 `path` 指定的地址（文件 / 目录）的文件信息，类型为 `os.FileInfo`。

如果遍历 `path` 指定的文件或目录时出现了问题，传入的参数 `err` 会描述该问题，`WalkFunc` 类型函数可以决定如何去处理该错误（`Walk` 函数将不会深入该目录）；如果该函数返回一个错误，`Walk` 函数的执行会中止；只有一个例外，如果 `Walk` 的 `walkFn` 返回值是 `SkipDir`，将会跳过该目录的内容而 `Walk` 函数照常执行处理下一个文件。

和 `os` 遍历目录树的示例对应，使用 `Walk` 遍历目录树的示例程序在  [walk](/code/src/chapter06/filepath/walk/main.go)，程序简单很多。

## Windows 起作用的函数

`filepath` 中有三个函数：`VolumeName`、`FromSlash` 和 `ToSlash`，针对非 Unix 平台的。

## 关于 path 包

`path` 包提供了对 `/` 分隔的路径的实用操作函数。

在 Unix 中，路径的分隔符是 `/`，但 Windows 是 `\`。在使用 `path` 包时，应该总是使用 `/`，不论什么系统。

`path` 包中提供的函数，`filepath` 都有提供，功能类似，但实现不同。

一般应该总是使用 `filepath` 包，而不是 `path` 包。
