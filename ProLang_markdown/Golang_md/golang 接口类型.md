# 接口

接口类型是对其它类型行为的抽象和概括；因为接口类型不会和特定的实现绑定在一起，通过这种抽象的方式我们可以让我们的函数更加灵活和更具有适应能力。

很多面向对象的语言都有相似的接口概念，但是 GO 语言中接口类型的独到之处在于它是满足隐式实现的。也就是说，我们没有必要对于给定的具体类型定义所有满足的接口类型；简单地拥有一些必要的方法就足够了。这种设计可以让你创建一个新的接口类型满足已经存在的具体类型却不会去改变这些类型的定义；当我们使用的类型来自于不受我们控制的包时这种设计尤其有用。

## 接口即约定

之前介绍的类型是具体类型。具体类型指定了它所含数据的精确布局，还暴露了基于这个精确布局的内部操作。

Go 语言中还有另外一种类型称为*接口类型*。接口类型是一种抽象类型，它并没有暴露所含数据的布局或者内部结构，当然也没有那些数据的基本操作。

fmt.Printf 和 fmt.Sprintf：前者吧结果发到标准输出（标准输出其实是一个文件），后者把结果以 string 类型返回。格式化是两个函数中最复杂的部分，如果仅仅因为两个函数在输出方式的轻微差异，就需要把格式化部分在两个函数中重复一遍。幸运的是没通过接口机制可以解决这个问题。两个函数迪封装了第三个函数 fmt.Fprintf，而这个函数结果实际输出到哪里毫不关心。

```golang
package fmt
func Fprintf(w io.Writer, format string, args ...interface{}) (int, error)
func Printf(format string, args ...interface{}) (int, error) {
    return Fprintf(os.Stdout, format, args...)
}
func Sprintf(format syring, args ...interface{}) string {
    var buf bytes.Buffer
    Fprintf(&buf, format, args...)
    return buf.String()
}
```

其实 Fprintf 的第一个形参也不是文件类型，而是 io.Writer 接口类型，其声明如下：

```golang
package io

// Writer 接口封装了基础的写入方法
type Writer interface {
    // Writer从p向底层数据流写入len(p)个字节的数据
    // 返回实际写入的字节数（0 <= n <= len(p)）
    // 如果没有写完，那么会返回遇到的错误
    // 在Write返回n<len(p)时，err必须为非nil
    // Write不允许p的数据，即使是临时修改
    //
    // 实现时不允许残留p的引用
    Write(p []byte) (n int, err error)
}
```

## 接口类型

一个接口类型定义了一套方法，如果一个具体类型的实现该接口，那么必须实现接口类型定义中的所有方法。

io.Writer 是一个广泛使用的接口，他负责所有的写入字节的类型的抽象，包括文件、内存缓冲区、网络连接、HTTP 客户端、打包器（archiver）、散列器（hasher）等。io 包还定义了很多有用的接口。Reader 就抽象了所有可读字节的类型，Closer 抽象了所有可以关闭的类型，比如文件或者网络连接。

```golang
package io
type Reader interface {
    Read(p []byte) (n int, err error)
}
type Closer interface {
    Close() error
}

// 另外，我们还可以发现通过组合已有的接口得到的新接口。

type ReadWriter interface {
    Reader
    Writer
}
type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

如何的语法称为嵌入式接口，与嵌入式结构类型，让我们可以直接使用一个接口，而不用注意写出这个接口所包含的方法。

```golang
// io.ReadWriter:
type ReadWriter interface {
    Read(p []byte) (n int, err error)
    Write(p []byte) (n int, err error)
}
// 也可以混合使用两种方法
type ReadWriter interface {
    Read(p []byte) (n int, err error)
    Write
}
```

三种声明的效果都是一致的。方法定义的顺序也是无意义的，真正有意义的只有接口的方法集合。

## 实现接口

如果一个类型实现了一个接口要求的所有方法，那么这个类型实现了这个接口。比如\*os.File 类型实现了 io.Reader、Writer、Closer 和 ReadWriter 接口。\*byte.Buffer 实现了 Reader、Writer 和 ReadWriter，但是没有实现 Closer，因为它没有 Close 方法。为了简化表述，Go 程序员通常说一个具体类型“是一个(is-a)”特定的接口类型，这其实代表着该具体类型实现了该接口。比如，*bytes.Buffer 是一个 io.Writer；*os.File 是一个 io.ReaderWriter。

接口的赋值规则很简单，仅当一个表达式实现了一个接口时，这个表达式才可以赋给该接口。

空接口类型的 interface{}是不可缺少的。看起来这个接口没有任何用途，正因为空接口类型对其实现类型没有任何要求，所以我们可以把任何值赋给空接口类型。

当然，即使我们创建了一个指向布尔值、浮点数、字符串、map、指针或其他类型的 interface{}接口，也无法直接使用其中的值，毕竟这个接口不包含任何方法。

判断是否实现接口只需要比较具体类型和接口类型的方法，所以没有必要再具体类型的定义中声明这种关系。

一个指向结构的指针才是最常见的方法接收者。

指针类型肯定不是实现接口的唯一类型，即使是那些包含了会改变接收者方法的接口类型，也可以由 GO 的其他引用类型来实现，

从具体类型出发、提取其共性而得到的每一种分组方式都可以表示为一种接口类型。与基于类的语言（他们显式地声明了一个类型实现的所有接口）不同的是，在 Go 语言里我们可以在需要时才定义新的抽象和分组，并不用修改原有类型的定义。当需要使用另一个作者写的包里的具体类型时，这一点特别有用。

## 接口值

从概念上来讲，一个接口类型的值（简称接口值）其实分为两部分：一个具体类型和改类型的值。二者称为接口动态类型和动态值。

对于像 Go 这样的静态类型语言，类型仅仅是一个编译时的概念，所以类型不是一个值。在我们的概念模型中，用类型描述符提供么个类型的具体信息。对于一个接口值，类型部分就用对应的类型描述符来表达。
