---
layout: blog
istop: true
title: "golang for循环之迭代变量"
background: yellow
background-image: https://fishingfly.github.io/style/images/music.jpg
date:  2020-01-17
category: golang
tags:
- golang
- for循环
- 迭代变量

---

面试题：

```go
package main
const N = 3
func main() {
	m := make(map[int]*int)
	for i := 0; i < N; i++ {
		m[i] = &i //A
	}
	for _, v := range m {
		print(*v)
	}
}
```

自己想一下结果，想一下为什么？解决方法是啥？

这边涉及的知识点是for循环中的迭代变量

其实很简单 引入一个临时局部变量就可以了，这样就可以将每次的值存储到该变量地址上

你可能会很奇怪为什么会出现这种情况，结果不应该是 `1, 4, 9, 16, 25` 吗？其实原因是循环变量的作用域的规则限制。在上面的程序中，`v` 在 for 循环引进的一个块作用域内进行声明。在循环里创建的所有函数变量`共享相同的变量，就是一个可访问的存储位置，而不是固定的值`。(你会惊奇的发现 `&v` 的内存地址是一样的)

模拟一下实际的情况，假设 v 变量的地址在 0x12345678 上， for 循环在迭代过程中，所有变量值都是在这地址上迭代的。当最后调用匿名函数的时候，取值也是在这块地址上。所以最后输出的结果都是迭代的最后一个值。至少在 Go 语言中是不用质疑的。这里也是一个陷阱，如果你不清楚的话，肯定会遇到坑。那个该如何修改呢？

```go
for i, v := range
```

- `i`,`v`都是只创建一次，然后循环中赋值。
- 循环的 Map，`golang`为避免开发者循环时问题，所以特意在循环中打乱排序
- 循环的数组，是在开始前的镜像，循环中添加或移除元素不改变其循环次数。
- 循环的 Map，由于其随机特性，循环中添加或移除元素不能确定是否改变循环次数





```go
package main
const N = 3
func main() {
	m := make(map[int]*int)
	for i := 0; i < N; i++ {
		j := int(i)
		m[i] = &j
	}
	for _, v := range m {
		print(*v)
	}
}
```



question2:下面程序的打印结果是什么

```go
package main

import (
	"fmt"
	"time"
)

type field struct {
	name string
}

func (p *field) print() {
	fmt.Println(p.name)
}

func main() {
	data := []field{{"one"}, {"two"}, {"three"}}
	for _, v := range data {
		go v.print()
	}
	time.Sleep(3 * time.Second)
}
```

打印结果：

three
three
three

那再看看下面程序的执行结果：

```go
package main

import (
	"fmt"
	"time"
)

type field struct {
	name string
}

func (p *field) print() {
	fmt.Println(p.name)
}

func main() {
	//data := []field{{"one"}, {"two"}, {"three"}}
	//for _, v := range data {
	//	go v.print()
	//}
	//time.Sleep(3 * time.Second)
	data := []*field{{"one"}, {"two"}, {"three"}}
	for _, v := range data {
		go v.print()
	}
	time.Sleep(3 * time.Second)
	//goroutines print: one, two, three
}
```

结果：

one
two
three

以下摘自：https://studygolang.com/topics/3775 中以为网友的评论，讲的很不错

*虽然楼主已经知道原因了，但是原因最好还是注明下，方便解惑他人。*

*先声明为了方便起见，我把指针称为地址，这样会更好理解一些。*

*先说下案例一 问题的关键在于print方法的声明，现在是func (p field) print()。 1、首先明确一点，在for循环结束之后，复用变量v的值其实已经变成了three。 2、其次注意print方法的声明，func (p field) print()，细心的同学发现了，print方法的接收者其实是一个地址。 3、for循环中，我们传递给print的方法接收者是对象v,但是方法接收者却要求是一个地址，这时，golang自动做了转换，把对象的地址传递过去了，注意是对象的地址传递过去了，而不是对象的值，参考第一条，因为v是复用变量，v的地址是不会变，但v的值会变，由one变成two，由two变成three，既然我们3次传递的都是v的地址，所以打印出来的自然是同一个值*

*再说下案例二 案例二中，print方法的声明依然是func (p *field) print(),但是打印出来的却是one,two,three,这是为何呢？ 原因就是在于这次我们传递不再是一个对象，而是一个对象地址，这个时候，我们传递的数据类型和方法声明的类型是同一个类型，这个时候，可以理解为golang不会对参数进行转换，而是直接拷贝了传递过来的地址，既然每次传递过来的地址都是不一样的，那么打印出来的值自然不同*

*如何让案例一也输出one,two,three呢？ 修改print方法的声明，func (p *field) print()改成func (p field) print()，我们把方法接收者改成对象，而不是对象的地址，,因为for中，每次传递过去的值都是不一样的，这个时候，打印出来的自然就是one,two,three。不建议这么做，原因在下面会说明。*

*说了那么多，总结一句话，注意传递给方法的接收者类型是不是该方法声明的类型，不一样的话，自然会出现一些意想不到的效果，最好在编码的时候，注意这点。*

*还有一点说明下上面为什么说不建议修改print方法的声明，案例一中，for循环其实是在拷贝数组中的每一个元素给v，这样内存是默默的一次次的无情消耗，所以，最好for循环遍历是指针数组，而不是对象数组。最好的方式是data就是一个指针数组，这样和方法声明一致。而且for循环中也不会产生大量的隐形拷贝行为。*

另外，我在看Golang相关资料是也看到关于值方法个指针方法的资源，当然我博客里也有讲到这方面的博文（），这边需要讲一下值方法和指针方法之间有什么不同点呢？它们的不同如下所示（以下摘自极客时间《Golang核心36讲》的关于指针的有限操作）。

1、值方法的接收者是该方法所属的那个类型值的一个副本。我们在该方法内对该副本的修改一般都不会体现在原值上，除非这个类型本身是某个引用类型（比如切片或字典）的别名类型。而指针方法的接收者，是该方法所属的那个基本类型值的指针值的一个副本。我们在这样的方法内对该副本指向的值进行修改，却一定会体现在原值上。

2、一个自定义数据类型的方法集合中仅会包含它的所有值方法，而该类型的指针类型的方法集合却囊括了前者的所有方法，包括所有值方法和所有指针方法。严格来讲，我们在这样的基本类型的值上只能调用到它的值方法。但是，Go 语言会适时地为我们进行自动地转译，使得我们在这样的值上也能调用到它的指针方法。转译过程就是先取该值的指针值，然后再该指针上调用指针方法。