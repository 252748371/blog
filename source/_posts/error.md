---
title: Go 异常处理机制——defer, error, panic, recover
date: 2018-07-27 15:00:17
tags: Go
---
### 1. 如果项目中的代码、使用的标准库以及第三方库在运行时内部捕获了异常并通过合适的 error 对象返回给调用者，那我们可以尽量少甚至可以不用 panic 函数。
### 2. 如果无法保证上面的情况，那为了确保程序在运行时不会因为 未知的未知 导致崩溃，那 panic 函数的使用可能不得不加在任何需要的地方。
### 3. 我们应该认识到，panic是我们和计算机都不希望看到的，应该在设计开发的时候充分考虑使用场景可能出现的情况，处理好 已知的未知，在确定需要的地方使用 panic 机制。

# defer 

defer关键字用来标记最后执行的Go语句，一般用在资源释放、关闭连接等操作，会在函数关闭前调用。

多个defer的定义与执行类似于栈的操作：先进后出，最先定义的最后执行。

<!-- more -->

请先看下边几段代码，然后判断一下各自输出内容：
```
// 示例代码一：
func funcA() int {
    x := 5
    defer func() {
        x += 1
    }()
    return x
}
// 示例代码二：
func funcB() (x int) {
    defer func() {
        x += 1
    }()
    return 5
}

// 示例代码三：
func funcC() (y int) {
    x := 5
    defer func() {
        x += 1
    }()
    return x
}
 
// 示例代码四：
func funcD() (x int) {
    defer func(x int) {
        x += 1
    }(x)
    return 5
}
```
解析这几段代码，主要需要理解清楚以下几点知识：
### 1. return语句的处理过程

return xxx 语句并不是一条原子指令，其在执行的时候会进行语句分解成 返回变量=xxx return，最后执行return

### 2. defer语句执行时机

上文说过，defer语句是在函数关闭的时候调用，确切的说是在执行return语句的时候调用，注意，是return 不是return xxx

### 3. 函数参数的传递方式

Go语言中普通的函数参数的传递方式是值传递，即新辟内存拷贝变量值，不包括slice和map，这两种类型是引用传递

### 4. 变量赋值的传递方式

Go语言变量的赋值跟函数参数类似，也是值拷贝，不包括slice和map，这两种类型是内存引用

按照以上原则，解析代码：
```
// 解析代码一：返回temp的值，在将x赋值给temp后，temp未发生改变，最终返回值为5
func funcA() int {
    x := 5
    temp=x      #temp变量表示未显示声明的return变量
    func() {
        x += 1
    }()
    return
}

// 解析代码二：返回x的值，先对其复制5，接着函数中改变为6，最终返回值为6
func funcB() (x int) {
    x = 5
    func() {
        x += 1
    }()
    return
}

// 解析代码三：返回y的值，在将x赋值给y后，y未发生改变，最终返回值为5
func funcC() (y int) {
    x := 5
    y = x       #这里是值拷贝
    func() {
        x += 1
    }()
    return
}

// 解析代码四：返回x的值，传递x到匿名函数中执行时，传递的是x的拷贝，不影响外部x的值，最终返回值为5
func funcD() (x int) {
    x := 5
    func(x int) { #这里是值拷贝
        x += 1
    }(x)
    return
}
```
# Error
Go语言 通过支持多返回值，让在运行时返回详细的错误信息给调用者变得非常方便。我们可以在编码中通过实现 error 接口类型来生成错误信息，error 接口的定义如下：
```
type error interface {
    Error() string
}
```
还是通过下面的例子来看看：
```
package main

import (
	"fmt"
)

// 定义一个 DivideError 结构
type DivideError struct {
	dividee int
	divider int
}

// 实现 	`error` 接口
func (de *DivideError) Error() string {
	strFormat := `
	Cannot proceed, the divider is zero.
	dividee: %d
	divider: 0`
	return fmt.Sprintf(strFormat, de.dividee)
}

// 定义 `int` 类型除法运算的函数
func Divide(varDividee int, varDivider int) (result int, errorMsg string) {
	if varDivider == 0 {
		dData := DivideError{
			dividee: varDividee,
			divider: varDivider,
		}
		errorMsg = dData.Error()
		return
	} else {
		return varDividee / varDivider, ""
	}

}

func main() {

	// 正常情况
	if result, errorMsg := Divide(100, 10); errorMsg == "" {
		fmt.Println("100/10 = ", result)
	}
	// 当被除数为零的时候会返回错误信息
	if _, errorMsg := Divide(100, 0); errorMsg != "" {
		fmt.Println("errorMsg is: ", errorMsg)
	}

}
```
运行后可以看到下面的输出：
```
100/10 =  10
errorMsg is:  
	Cannot proceed, the divider is zero.
	dividee: 100
	divider: 0
```
# Panic 和 recover
定义如下：
```
func panic(interface{})
func recover() interface{}
```
panic 和 recover 是两个内置函数，用于处理 run-time panics 以及程序中自定义的错误。

当执行一个函数 F 的时候，如果显式地调用 panic 函数或者一个 run-time panics 发生时，F 会结束运行，所有 F 中 defer 的函数会按照 FILO 的规则被执行。之后，F 函数的调用者中 defer 的函数再被执行，如此一直到最外层代码。这时，程序已经被中断了而且错误也被一层层抛出来了，其中包括 panic 函数的参数。当前被中断的 goroutine 被称为处于 panicking 状态。由于 panic 函数的参数是空接口类型，所以可以接受任何类型的对象：
```
panic(42)
panic(42)
panic("unreachable")
panic(Error("cannot parse"))
```
recover 函数用来获取 panic 函数的参数信息，只能在延时调用 defer 语句调用的函数中直接调用才能生效，如果在 defer 语句中也调用 panic 函数，则只有最后一个被调用的 panic 函数的参数会被 recover 函数获取到。如果 goroutine 没有 panic，那调用 recover 函数会返回 nil。
```
package main

import (
	"fmt"
)

// 最简单的例子
func SimplePanicRecover() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Println("Panic info is: ", err)
		}
	}()
	panic("SimplePanicRecover function panic-ed!")
}

// 当 defer 中也调用了 panic 函数时，最后被调用的 panic 函数的参数会被后面的 recover 函数获取到
// 一个函数中可以定义多个 defer 函数，按照 FILO 的规则执行
func MultiPanicRecover() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Println("Panic info is: ", err)
		}
	}()
	defer func() {
		panic("MultiPanicRecover defer inner panic")
	}()
	defer func() {
		if err := recover(); err != nil {
			fmt.Println("Panic info is: ", err)
		}
	}()
	panic("MultiPanicRecover function panic-ed!")
}

// recover 函数只有在 defer 函数中被直接调用的时候才可以获取 panic 的参数
func RecoverPlaceTest() {
	// 下面一行代码中 recover 函数会返回 nil，但也不影响程序运行
	defer recover()
	// recover 函数返回 nil
	defer fmt.Println("recover() is: ", recover())
	defer func() {
		func() {
			// 由于不是在 defer 调用函数中直接调用 recover 函数，recover 函数会返回 nil
			if err := recover(); err != nil {
				fmt.Println("Panic info is: ", err)
			}
		}()

	}()
	defer func() {
		if err := recover(); err != nil {
			fmt.Println("Panic info is: ", err)
		}
	}()
	panic("RecoverPlaceTest function panic-ed!")
}

// 如果函数没有 panic，调用 recover 函数不会获取到任何信息，也不会影响当前进程。
func NoPanicButHasRecover() {
	if err := recover(); err != nil {
		fmt.Println("NoPanicButHasRecover Panic info is: ", err)
	} else {
		fmt.Println("NoPanicButHasRecover Panic info is: ", err)
	}
}

// 定义一个调用 recover 函数的函数
func CallRecover() {
	if err := recover(); err != nil {
		fmt.Println("Panic info is: ", err)
	}
}

// 定义个函数，在其中 defer 另一个调用了 recover 函数的函数
func RecoverInOutterFunc() {
	defer CallRecover()
	panic("RecoverInOutterFunc function panic-ed!")
}

func main() {
	SimplePanicRecover()
	MultiPanicRecover()
	RecoverPlaceTest()
	NoPanicButHasRecover()
	RecoverInOutterFunc()
}
```
运行后可以看到下面的输出：
```
Panic info is:  SimplePanicRecover function panic-ed!
Panic info is:  MultiPanicRecover function panic-ed!
Panic info is:  MultiPanicRecover defer inner panic
Panic info is:  RecoverPlaceTest function panic-ed!
recover() is:  <nil>
NoPanicButHasRecover Panic info is:  <nil>
Panic info is:  RecoverInOutterFunc function panic-ed!
```
