+++
date = '2026-09-23T10:36:31+08:00'
draft = false
title = 'Go 语言中的循环控制'
categories = ['编程']
tags = ['Go', '基础知识', '流程控制', 'For']
+++

Go 语言只有一种循环构造方式，即 for 循环（**没有 while 或 do-while**），类似于 C、Java 和 C# 之类的编程语言中的 for 循环。使用分号`;`分隔。。它的基本语法如下：
```go
for 初始化语句; 条件表达式; 后置语句 {
    循环体
}
```
**执行顺序**
1. 初始化语句：在循环开始前执行，且只执行一次（如 `i := 0`）。
2. 条件表达式：在每次迭代前求值。如果为`true`，进入循环体；如果为`false`，循环直接结束。
3. 循环体：条件为真时执行。
4. 后置语句：在每次循环体执行结束后执行（如 `i++`），然后回到第 2 步重新判断条件。


# 1. 基本语法
**for循环的三个组件**：
- 在第一次迭代之前执行的初始语句（可选）。
- 在每次迭代之前计算的条件表达式，该条件为false时，循环会停止退出。
- 在每次迭代结束时执行的后处理语句（可选）。


```go
func main() {
    sum := 0
    for i := 1; i <= 100; i++ {
        sum += i
    }
    fmt.Println("sum of 1..100 is", sum)
}
```


# 2. 只使用表达式的 for 循环
某些编程语言中，可以使用 while 关键字编写循环，而 Go 没有 while 关键字。 但是可以改用 for 循环实现相同的效果。

```go
package main

import (
    "fmt"
    "math/rand"
    "time"
)

func main() {
    var num int64
    rand.Seed(time.Now().Unix())
    for num != 5 {
        num = rand.Int63n(15)
        fmt.Println(num)
    }
}
```

# 3. 无限循环和 break
更进一步，当 for 循环连表达式都不编写的情况就是无限循环，这时想要退出循环必须使用 break 关键字。

```go
package main

import (
    "fmt"
    "math/rand"
    "time"
)

func main() {
    var num int32
    sec := time.Now().Unix()
    rand.Seed(sec)

    for {
        fmt.Print("Writting inside the loop...")
        if num = rand.Int31n(10); num == 5 {
            fmt.Println("finish!")
            break
        }
        fmt.Println(num)
    }
}
```
**注意**：Go 语言规范中明确规定，不带 label 的 break 语句中断执行并跳出的，是同一函数内 break 语句所在的最内层的 for, switch, select。因此当上述三种语句搭配或嵌套使用时，break 仅能跳出当前的内层循环。

>带 label 的 break 语句使用下面的第 5 小节。

# 4. Continue 语句
和其他语言一样，可以使用 continue 关键字跳过循环的当前迭代。

```go
package main

import "fmt"

// 求出 1-100 中不能被 5 整除的所有数的和。
func main() {
    sum := 0
    for num := 1; num <= 100; num++ {
        if num%5 == 0 {
            continue
        }
        sum += num
    }
    fmt.Println("The sum of 1 to 100, but excluding numbers divisible by 5, is", sum)
}
```

# 5. label
Go 语言中的 break 和 continue 关键字还有一种搭配 label 来使用的方式，看下面的例子：

```go
package main

import "fmt"

func labelContinue() {
	var arr2 = [][]int{
		{1, 4, 6, 7, 9},
		{2, 5, 7, 9, 4},
		{3, 5, 1, 6, 7},
	}

outerloop:
	for i := 0; i < len(arr2); i++ {
		for j := 0; j < len(arr2[i]); j++ {
			if arr2[i][j] == 7 {
				fmt.Printf("found 7 at [%d, %d]\n", i, j)
				continue outerloop
			}
		}
	}
}

func main() {
	labelContinue()
}
```

在上面的代码中，我们定义了 arr2 的二维数组，然后使用嵌套的 for 循环想找出二维数组中所有元素 7 所在的位置，由于内层的一位数组有且只有一个元素 7，因此，在找到元素 7 后就不想继续遍历了，以此来提高代码效率。

在其它语言中通常的做法可能是使用 break 关键字来中断内层循环，而 Go 提供了一种更加独特对方式：`continue + label`。可以看到在最外层的循环上面我们定义了一个名为 outerloop 的 label，然后在内层循环中使用`continue + label`，这样的写法会使代码从指定的 label 处继续执行，从而实现让代码跳转到最外层的循环。

接下来我们再看一个 break 搭配 label 使用的例子：
```go
package main

import "fmt"

func labelBreak() {
	var arr2 = [][]int{
		{1, 4, 6, 7, 9},
		{2, 5, 3, 9, 4},
		{0, 5, 1, 6, 7},
	}

outerloop:
	for i := 0; i < len(arr2); i++ {
		for j := 0; j < len(arr2[i]); j++ {
			if arr2[i][j] == 3 {
				fmt.Printf("found 7 at [%d, %d]\n", i, j)
				break outerloop
			}
		}
	}
}

func main() {
	labelBreeak()
}
```
整体代码和之前类似，所不同的是这次在整个二位数组中有且只有一个元素 3，因此当我们找到这个元素时，就可以直接终止最外层循环。

而前面我们提到过，单独使用 break 关键字只能退出当前循环，因此 Go 语言提供了 break + label 的语法，用来终止指定 label 处的循环。

# 6. range 关键字
for 循环的 range 格式可以对 slice、map、数组、字符串、channel 等进行迭代循环。

`for range`遍历的返回值遵循以下规律：
- 数组、切片、字符串返回索引和值。
- map 返回键和值。
- channel 只返回通道内的值。

```go
package main

import "fmt"

func main() {
	s := []int{1, 3, 5}

	for index, value := range s {
		fmt.Printf("index=%d value=%d\n", index, value)
	}
}

// 输出
index=0 value=1
index=1 value=3
index=2 value=5
```

对于`for range`返回的下标和值，如果我们不关心元素的值，可以省略变量 value：
```go
for index := range s {
	fmt.Printf("index=%d", index)
}
```

如果我们不关心下标，只需要元素值，可以使用空标识符`_`代替下标变量 index：
```go
for _, value := range s {
	fmt.Printf("value=%d", value)
}
```

更极端一些，我们既不关心下标值，也不关心元素值，可以使用两个空标识符`_`来接收变量，但 go 官方考虑到这样的代码看起来不太优雅，提供了一种等价的写法：
```go
for range s {
	fmt.Printf("i don't need index and value.")
}
```

