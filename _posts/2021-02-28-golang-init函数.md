---
layout: mypost
title: Golang init函数
categories: [Go, Golang]
---

转载自https://zhuanlan.zhihu.com/p/88128042  作者: 开源小喵Airth

Golang 的init函数和其他函数或方法有诸多不同. 它是 Golang package 初始化中使用的重要角色, 可以说是语法糖. 当对于 Golang 这样一门工程化编程语言来说,init函数有着很多巧妙的使用. 本文从init函数的的一些特性开始, 并附加部分标准库中的例子, 来谈谈init函数的使用方法.

1. 不唯一性
   init函数和其他函数最大的区别之一就是, 同一个 package 或源文件中, 可以有很多个. 我们看下面的例子:

package main

import (
"fmt"
)

func init() {
fmt.Println("init 2")
}

func init() {
fmt.Println("init 1")
}

func main() {
fmt.Println("main")
}
运行结果:

init 2
init 1
main
如果同一个 Package 下的多个源文件中都有init函数, 也是没有问题的.

2. 生命周期
   init is called after all the variable declarations in the package have evaluated their initializers, and those are evaluated only after all the imported packages have been initialized. Link
   init函数在一个 package 中的所有全局变量都初始化完成之后, 才开始运行. 这一点非常方便代码的组织. 例如当一个package 中有非常多的方法或函数, 这些方法逻辑上都处于同一个级别, 进一步拆分 package 并不合理. 这时候我们就可以将这些方法或函数放在多个源文件中. 而对这些方法或函数初始化的init函数可以和对应的逻辑放在一起, 这也能体现代码设计上的 Cohesion.

例如在一个名为api的 package 中:

./api/
├── account.go
├── profile.go
├── resource.go
└── user.go
API 对应的初始化部分都可以单独地写在每个源文件中, 引用这个 package 的开发者并不需要显示地调用初始化函数就能完成整个 package 的初始化.

其次 init 函数只会运行一次, 即使被 import 了很多次.

Package initialization is done only once even if package is imported many times.
这对很多需要维护全局唯一的一些特性非常有用, 例如在流行的日志工具logrus的实现中, 为了提升获取时间的性能, logrus 在初始化的时候获取了系统的基准时间, 而这个时间需要全局唯一, 并且只需要获取一次. 代码如下:

// Link https://github.com/sirupsen/logrus/blob/d5d4df1108f606433e95b17c8fbc110916779780/text_formatter.go#L26

package logrus

import (
"time"
)

var baseTimestamp time.Time

func init() {
baseTimestamp = time.Now()
}
3. 没有输入输出的参数
   init function is niladic. Link
   如果我们给init函数写上输入参数或输出参数会怎么样呢?init函数会不会变成一个普通的函数? 答案是:

func init must have no arguments and no return values.
Compiler 会告诉我们, 这样写是语法错误的, 这也说明了init函数在 Golang 语法体系中的特殊性.

4. 运行顺序
   同一个源文件中, 写在更靠近文件上面的 init 函数更早运行
   同一个 package 中, 文件名排序靠前的文件中的 init 函数更早运行
   第一条没有什么疑问, 对于第二条, 可以简单地参考字符串比较, 例如:

a.go > b.go
a1.go > a2.go
5. 用作 side effect
   标准库中的 MySQL Driver 就是通过导入一个匿名的 package 来实现 side effect. 例如:

import "database/sql"
import _ "github.com/go-sql-driver/mysql"

db, err := sql.Open("mysql", "user:password@/dbname")
在上面的代码中, 我们导入了 MySQL 的 Driver, 却没有显示地使用它, 那这行导入实际上发生了什么呢, 如果我们去看 go-sql-driver/mysql 这个 package 的实现, 就会发现:

// Link: https://github.com/go-sql-driver/mysql/blob/578c4c8066964679ef44f45de2b6c7e811cc665e/driver.go#L83

func init() {
sql.Register("mysql", &MySQLDriver{})
}
这里在init函数中调用 sql 的标准接口, 注册了 MySQL 的 Driver. 这样做的好处是开发者并不需要显示地调用初始化函数.

最后, 标题就算小结了. 再提供一个 Tips:init函数并不一定需要写在源文件的最上面, 从语法层面说, 写在任何地方都可以. 标准库里有很多实践的例子. 但从实用和工程化的考量, 可以总结为已下思路:

可以将 init 函数写在和 init 函数初始化的内容相关的函数上面(特别是有多个 init 函数的情况下)
如果没有特别想关的内容, init 函数就放在源文件的最上面或最下面(方便被看到)
如果一个 package 只有一个init函数, 那尽量放在和 package 同名的源文件里, 例如 go-sql-driver/mysql 就放在mysql.go这个文件中