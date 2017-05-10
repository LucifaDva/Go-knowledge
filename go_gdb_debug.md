# 使用GDB调试golang代码

============

环境准备
---------------

1.代码demo.go

    任意demo.go代码，比如hello world:

    package main

	import (
        "fmt"
	)

	func main() {
        fmt.Println("hello world")
	}

2.进行编译：

    删除调试符号：go build -ldflags “-s -w”
    - -s:去掉符号信息
    - -w:去掉DWARF调试信息

    关闭内联优化：go build -gcflags “-N -l”

3.进行调试

sudo gdb demo

(gdb) source /usr/local/Cellar/go/1.8.1/libexec/src/runtime/runtime-gdb.py
Loading Go Runtime support.
(gdb) 