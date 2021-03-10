```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, Word!")
}
```

`package main`是声明这是一个独立可执行程序，而不是被外部引用的包。相反，`import "fmt"`则是从外部引入`fmt`包。

main()函数是程序的入口。



- `main package`不同于其它`library package`，它定义了一个可执行程序。其中的`main`函数即是可执行文件的入口函数。
- main函数默认是每一个可执行程序的入口
- package则用于包装和组织相关的函数、变量和常量。在使用一个package之前，我们需要使用import语句导入包。例如，我们这个程序中导入了fmt包（fmt是format单词的缩写，表示格式化相关的包），然后我们才可以使用fmt包中的Println函数。