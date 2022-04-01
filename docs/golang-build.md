# go build 
> 首先前提 go打包默认是采用静态打包的（所以打出的包比较大）
  打包后可以用ldd命令 判断是否有包依赖
## 静态链接

Go将所有运行需要的函数代码都放到了hellogo中，这就是所谓的“静态链接”

默认情况下，Go的runtime环境变量CGO_ENABLED=1，即默认开始cgo，如果依赖了C代码的可能默认go build会依赖到一些包

所以如果要完全静态链接可以禁用CGO

但是禁用CGO 有时候会让编译失败  

问题来了：在CGO_ENABLED=1这个默认值的情况下，是否可以实现纯静态连接呢？
### internal linking和external linking  
答案是可以。在$GOROOT/cmd/cgo/doc.go中，文档介绍了cmd/link的两种工作模式：internal linking和external linking。

1、internal linking
internal linking的大致意思是若用户代码中仅仅使用了net、os/user等几个标准库中的依赖cgo的包时，cmd/link默认使用internal linking，而无需启动外部external linker(如:gcc、clang等)，不过由于cmd/link功能有限，仅仅是将.o和pre-compiled的标准库的.a写到最终二进制文件中。因此如果标准库中是在CGO_ENABLED=1情况下编译的，那么编译出来的最终二进制文件依旧是动态链接的，即便在go build时传入-ldflags '-extldflags "-static"'亦无用，因为根本没有使用external linker：

```golang
go build -o server-fake-static-link  -ldflags '-extldflags "-static"' server.go

```
2、external linking
而external linking机制则是cmd/link将所有生成的.o都打到一个.o文件中，再将其交给外部的链接器，比如gcc或clang去做最终链接处理。如果此时，我们在cmd/link的参数中传入-ldflags '-linkmode "external" -extldflags "-static"'，那么gcc/clang将会去做静态链接，将.o中undefined的符号都替换为真正的代码。我们可以通过-linkmode=external来强制cmd/link采用external linker，还是以server.go的编译为例：

```golang go build -o server-static-link  -ldflags '-linkmode "external" -extldflags "-static"' server.go
# command-line-arguments
/Users/tony/.bin/go18/pkg/tool/darwin_amd64/link: running clang failed: exit status 1
ld: library not found for -lcrt0.o
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```
可以看到，cmd/link调用的clang尝试去静态连接libc的.a文件，但由于我的mac上仅仅有libc的dylib，而没有.a，因此静态连接失败。我找到一个ubuntu 16.04环境：重新执行上述构建命令：

```golang go build -o server-static-link  -ldflags '-linkmode "external" -extldflags "-static"' server.go
 ldd server-static-link
    not a dynamic executable
 nm server-static-link|grep " U "
 ```
该环境下libc.a和libpthread.a分别在下面两个位置：

/usr/lib/x86_64-linux-gnu/libc.a
/usr/lib/x86_64-linux-gnu/libpthread.a
就这样，我们在CGO_ENABLED=1的情况下，也编译构建出了一个纯静态链接的Go程序。

如果你的代码中使用了C代码，并依赖cgo在go中调用这些c代码，那么cmd/link将会自动选择external linking的机制：
```golang
//testcgo.go
package main

//#include 
// void foo(char *s) {
//    printf("%s\n", s);
// }
// void bar(void *p) {
//    int *q = (int*)p;
//    printf("%d\n", *q);
// }
import "C"
import (
    "fmt"
    "unsafe"
)

func main() {
    var s = "hello"
    C.foo(C.CString(s))

    var i int = 5
    C.bar(unsafe.Pointer(&i))

    var i32 int32 = 7
    var p *uint32 = (*uint32)(unsafe.Pointer(&i32))
    fmt.Println(*p)
}
```
编译testcgo.go：

```golang 
go build -o testcgo-static-link  -ldflags \'-extldflags "-static"\' testcgo.go
 ldd testcgo-static-link
    not a dynamic executable
```
vs.
```golang 
 go build -o testcgo testcgo.go
 ldd ./testcgo
    linux-vdso.so.1 =>  (0x00007ffe7fb8d000)
    libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007fc361000000)
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fc360c36000)
    /lib64/ld-linux-x86-64.so.2 (0x000055bd26d4d000)
```
小结


你的程序用了哪些标准库包？如果仅仅是非net、os/user等的普通包，那么你的程序默认将是纯静态的，不依赖任何c lib等外部动态链接库；
如果使用了net这样的包含cgo代码的标准库包，那么CGO_ENABLED的值将影响你的程序编译后的属性：是静态的还是动态链接的；
CGO_ENABLED=0的情况下，Go采用纯静态编译；
如果CGO_ENABLED=1，但依然要强制静态编译，需传递-linkmode=external给cmd/link。

https://johng.cn/cgo-enabled-affect-go-static-compile/#internal_linkingexternal_linking
