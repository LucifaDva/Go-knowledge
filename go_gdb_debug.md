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

    删除调试符号：go build -ldflags "-s -w"
    - -s:去掉符号信息
    - -w:去掉DWARF调试信息

    关闭内联优化：go build -gcflags "-N -l"

3.进行调试

sudo gdb demo

(gdb) source $GOROOT/src/runtime/runtime-gdb.py
Loading Go Runtime support.
// 查看代码（1）
(gdb) l main.main
2	
3	import (
4		"fmt"
5	)
6	
7	func main() {
8		fmt.Println("hello world")
9	}
// 查看源码（2）
(gdb) l runtime.gcStart
925	// gcBackgroundMode) by performing sweep termination and GC
926	// initialization.
927	//
928	// This may return without performing this transition in some cases,
929	// such as when called on a system stack or with locks held.
930	func gcStart(mode gcMode, forceTrigger bool) {
931		// Since this is called from malloc and malloc is called in
932		// the guts of a number of libraries that might be holding
933		// locks, don't attempt to start GC in non-preemptible or
934		// potentially unstable situations.
// 回车继续
(gdb) 
935		mp := acquirem()
936		if gp := getg(); gp == mp.g0 || mp.locks > 1 || mp.preemptoff != "" {
937			releasem(mp)
938			return
939		}
940		releasem(mp)
941		mp = nil
942	
943		// Pick up the remaining unswept/not being swept spans concurrently
944		//
// 设置断点
(gdb) b main
Breakpoint 1 at 0x104d280: file $GOROOT/src/runtime/rt0_darwin_amd64.s, line 78.
(gdb) b main.main
Breakpoint 2 at 0x1087050: file ...demo.go, line 7.
(gdb) b runtime.newproc
Breakpoint 3 at 0x102e630: file $GOROOT/src/runtime/proc.go, line 2841.
(gdb) b runtime.newproc1
Breakpoint 4 at 0x102e6d0: file $GOROOT/src/runtime/proc.go, line 2853.
(gdb) b runtime.wakep
Breakpoint 5 at 0x102b8d0: file $GOROOT/src/runtime/proc.go, line 1774.
(gdb) b runtime.mstart
Breakpoint 6 at 0x102a280: file $GOROOT/src/runtime/proc.go, line 1132.
(gdb) b runtime.startm
Breakpoint 7 at 0x102b450: file $GOROOT/src/runtime/proc.go, line 1674.
(gdb) b runtime.execute
Breakpoint 8 at 0x102bcd0: file $GOROOT/src/runtime/proc.go, line 1866.
(gdb) b runtime.schedule
Breakpoint 9 at 0x102cba0: file $GOROOT/src/runtime/proc.go, line 2172.
(gdb) b runtime.gcStart
Breakpoint 10 at 0x10146c0: file $GOROOT/src/runtime/mgc.go, line 930.
(gdb) b runtime.exit
Breakpoint 11 at 0x104d290: file $GOROOT/src/runtime/sys_darwin_amd64.s, line 20.
// 查看设置的断点
(gdb) info breakpoints
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x000000000104d280 in main at $GOROOT/src/runtime/rt0_darwin_amd64.s:78
2       breakpoint     keep y   0x0000000001087050 in main.main at ...demo.go:7
3       breakpoint     keep y   0x000000000102e630 in runtime.newproc at $GOROOT/src/runtime/proc.go:2841
4       breakpoint     keep y   0x000000000102e6d0 in runtime.newproc1 at $GOROOT/src/runtime/proc.go:2853
5       breakpoint     keep y   0x000000000102b8d0 in runtime.wakep at $GOROOT/src/runtime/proc.go:1774
6       breakpoint     keep y   0x000000000102a280 in runtime.mstart at $GOROOT/src/runtime/proc.go:1132
7       breakpoint     keep y   0x000000000102b450 in runtime.startm at $GOROOT/src/runtime/proc.go:1674
8       breakpoint     keep y   0x000000000102bcd0 in runtime.execute at $GOROOT/src/runtime/proc.go:1866
9       breakpoint     keep y   0x000000000102cba0 in runtime.schedule at $GOROOT/src/runtime/proc.go:2172
10      breakpoint     keep y   0x00000000010146c0 in runtime.gcStart at $GOROOT/src/runtime/mgc.go:930
11      breakpoint     keep y   0x000000000104d290 in runtime.exit at $GOROOT/src/runtime/sys_darwin_amd64.s:20
12      breakpoint     keep y   0x000000000104d290 in runtime.exit at $GOROOT/src/runtime/sys_darwin_amd64.s:20
13      breakpoint     keep y   0x000000000104d290 in runtime.exit at $GOROOT/src/runtime/sys_darwin_amd64.s:20
14      breakpoint     keep y   0x000000000104d290 in runtime.exit at $GOROOT/src/runtime/sys_darwin_amd64.s:20
15      breakpoint     keep y   0x000000000104d290 in runtime.exit at $GOROOT/src/runtime/sys_darwin_amd64.s:20
16      breakpoint     keep y   0x000000000104d290 in runtime.exit at $GOROOT/src/runtime/sys_darwin_amd64.s:20
17      breakpoint     keep y   0x000000000104d290 in runtime.exit at $GOROOT/src/runtime/sys_darwin_amd64.s:20
18      breakpoint     keep y   0x000000000104d290 in runtime.exit at $GOROOT/src/runtime/sys_darwin_amd64.s:20
// 开始运行
(gdb) r
Starting program: demo 
warning: unhandled dyld version (15)

Breakpoint 1, main () at $GOROOT/src/runtime/rt0_darwin_amd64.s:78
78		MOVQ	$runtime·rt0_go(SB), AX
// c，继续运行...
(gdb) c
...
...
// 查看 goroutines 信息
(gdb) info goroutines
  1 runnable runtime.gcenable
  2 runnable runtime.forcegchelper
// 查看指定序号的 goroutine 调用堆栈
(gdb) goroutine 1 bt
#0  0x000000000101396a in runtime.gcenable () at $GOROOT/src/runtime/mgc.go:210
#1  0x000000000102744a in runtime.main () at $GOROOT/src/runtime/proc.go:151
#2  0x000000000104c3a1 in runtime.goexit () at $GOROOT/src/runtime/asm_amd64.s:2197
#3  0x0000000000000000 in ?? ()
// 堆栈帧信息
(gdb) info frame
...
// 查看局部变量
(gdb) info locals
p = 0x400000000
// 打印变量
(gdb) p p
$1 = (struct runtime.p *) 0x400000000
// 查看变量类型
(gdb) whatis p
type = struct runtime.p *
// 从参数信息中，我们可以看到命名返回参数的值
(gdb) info args
_p_ = 0x0
spinning = true


