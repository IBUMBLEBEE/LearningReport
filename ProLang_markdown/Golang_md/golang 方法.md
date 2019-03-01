# golang 方法

- 关键原则：封装和组合

## 方法声明

方法的的声明和普通函数的声明类似，只是在函数名字前面多了一个参数。**这个参数把这个方法绑定到这个参数对应的类型上。**

```golang
// Point 类型的方法
func (p Point) Distance(q Point) float64 {...}
```

附加的参数 p 称为参数的接收者，用来描述主调方法就像对象发送消息。

Go 语言中，接收者不适用特殊名（比如 this 或者 self）；而是我们自己选择接收者名字，就像其他的参数变量一样。由于接收者会频繁地使用，因此最好能够选择简短且在整个方法中名称始终保持一致的名字。最常见的方法就是取类型的名称的首字母，就像 Point 中的 p。

上面两个 Distance 函数声明没有冲突。第一个声明一个包级别的函数（称为 geometry.Distance）。第二个声明一个类型 Point 的方法，因此它的名字是 Point.Distance。

表达式 p.Distance 称作选择子（selector），因为它为接收者 p 选择合适的 Distance 方法。选择子也用与选择结构类型中的某些字段值，就像 p.x 中的字段值。**由于方法和字段来自于同一个命名空间，因此在 Point 结构类型中声明一个叫作 X 的方法会与字段 X 冲突，编译器会报错。**因为每个**类型**都有自己的命名空间，所以我们能够在其他不同类型中使用名字 Distance 作为方法名。

```golang
...

type Point struct {
	X, Y float64
}

func (pp Point) X(q Point) float64 {
	return math.Hypot(q.X-pp.X, q.Y-pp.Y)
}
...
// error info: type Point has both field and method named X
```

Go 和许多其他面向对象的语言不同，它可以将方法绑定到任何类型上。可以很方便地为简单的类型（如数字、字符串、slice、map，甚至函数等）定义附加的行为。同一个包下的任何类型都是可以声明方法，只要它的类型既不是指针类型也不是接口类型。

## 指针接收者的方法

由于驻点函数会复制每一个参数变量，如果函数需要更新一个变量，或者如果一个实参太大而我们希望避免复制整个实参，因此我们必须使用指针来传递变量的地址。

命名类型（Point）与指向它们的指针（\*Point）是唯一可以出现在接收者声明处的类型。为防止混淆，不允许本身是指针类型进行方法声明：

```golang
type P *int
func (P) func() { /* ... */} // 编译错误，非法的接收者类型
```

如果接收者 p 是 Point 类型的变量，但方法要求一个\*Point 接收者，我们可以使用简写：p.ScaleBy(2) 。实际上比那一起会对变量进行&p 的隐式转换。只有变量才允许这么做，包括结构体字段，像 p.X 和数组或者 slice 的元素，比如 perim[0]。不能够对一个不能取地址的 Point 类型接收者参数调用\*Point 方法，因为无法获取临时变量的地址。

**指针解引用**的意思的通过指针访问被指向的值。指针 a 的解引用表示为：\*a。

如果实参接收者是\*Point 类型，以 Point.Distance 的方式调用 Point 类型的方法是合法的，因为我们有办法从地址中获取 Point 的值；只要解引用指向接收者的指针值即可。编译器自动插入一个隐氏的 \* 操作符。下面两个函数的调用效果是一样的：

```golang
pptr.Distance(q)
(*pptr.Distance(q))
```

在合法的方法调用表达式中，只有符合下面 👇 三种形式的语句才能够成立。

- 实参接收者和形参接收者是同一类型，比如都是 T 类型或者\*T 类型：

  ```golang
  Point{1, 2}.Distance(q) // T类型是Point，q也是Point
  pptr.ScaleBy(2)  // 隐氏的*操作符，编译器翻译为（*pptr).ScaleBy(2) *Point类型，*指针取值操作，2是具体值，不是内存地址或者是Point
  ```

- 实参接收者是 T 类型的变量而形参接收者是\*T 类型。编译器会隐式地解引用接收者，获得实际的取值。

  ```golang
  p.ScaleBy(2) // 隐式转换为（&p）
  ```

- 实参接收者是\*T 类型而形参接受这是 T 类型。编译器会隐式地解引用接收者，获得实际的取值。

  ```golang
  pptr.Distance(q) // 隐式转换为（*pptr）
  ```

如果所有类型的 T 方法的接收者是 T 自己（而非\*T），那么复制它的实例安全的；调用方法的时候都必须进行一次复制。但是任何方法的接收者是指针的情况下，应该避免复制 T 的实例，因为这么做可能会破坏内部原本的数据。

## nil 是一个合法的接收者

就像一些函数允许 nil 指针作为实参，方法的接收者也一样，尤其是当 nil 是类型中有意义的零值时。

```golang
// IntList 是整型链表
// *IntList的类型nil代表空列表
type IntList struct {
	Value int
	Tail *IntList
}

func (list *IntList) sum() int {
	if list == nil {
		return 0
	}
	return list.Value + list.Tail.Sum()
}
```

当定义一个类型允许 nil 作为接收者时，应当在文档注释中显式地标明。

```golang
// Value 映射字符串到字符串列表
type Values map[string][]string

// Get 返回一个具体给定的Key的值
// 如不存在，则返回空字符串
func (v Values) Get(Key string) string {
	if vs := v[Key]; len(vs) > 0 {
		return vs[0]
	}
	return ""
}

// 添加一个键值到对应key列表中
func (v Values) Add(key, value string) {
	v[key] = append(v[key], value)
}
```

它的实现是 map 类型但也提供了一系列方法来简化 map 的操作，它的值是字符串 slice，即一个多重 map。使用者可以使用它固有的操作方式（make、slice 字面量、m[key]，等方法）。

## 通过结构体内嵌组成类型

匿名字段类型可以是个指向命名类型的指针，这个时候，字段和方法间接地来自于所指向的对象。这个可以让我们共享通用的结果以及对象之间的关系更加动态、多样化。

结构体可以拥有多个匿名字段。声明 ColorredPoint：

```golang
type ColoredPoint struct {
	Point
	color.RGBA
}
```

那么这个类型的值可以拥有 Point 所有的方法和 RGBA 所有的方法，以及任何其他直接在 ColoredPoint 类型中声明的方法。

方法只能在命名的类型（比如 Point）和指向它们的指针（\*Point）中声明，但内嵌帮助我们能够在未命名的结构体类型中声明方法。

```golang
var (
	mu sync.Mutex
	mapping = make(map[string]string)
)

func Lookup(key string) string {
	mu.Lock()
	v := mapping[key]
	mu.Unlock()
	return v
}
```

```golang
var cache = struct {
	sync.Mutex
	mapping map[string]string
} {
	mapping: make(map[string]string),
}

func Look(key string) string {
	cache.Lock()
	v := cache.mapping[key]
	cache.Unlock()
	return v
}
```

新的变量名更加贴切，而且 sync.Mutex 是内嵌的，他的 Lock 和 Unlock 方法也包含了结构体中，允许我们直接使用变量本身进行加锁。

## 方法变量与表达式

通常我们都在相同的表达式里使用和调用方法，就像在 p.Distance()中，但是把两个操作分开也是可以的。选择子 p.Distance 可以赋予一个方法变量，它是一个函数，把方法（Point.Distance）绑定到一个接收者 p 上。函数只需提供实参而不需要提供接收者就能够调用。

```golang
p := Point{1, 2}
q := Point{4, 6}

distanceFromP := p.Distance	   // 方法变量
fmt.Println(distanceFromP(q))  // 函数distanceFromP
var origin Point
fmt.Println(distanceFromP(origin))

scaleP := p.ScaleBy  // 方法变量
scaleP(2)            // p变成(2, 4)

```

如果包内的 API 调用一个函数值，并且使用者期望这个函数的行为是调用一个特定接收者的方法，方法变量就非常有用。

比如，函数 time.AfterFunc 会在指定的延迟后调用一个函数值。这个程序使用 time.AfterFunc 在 10s 后启动火箭 r：

```golang
type Rocket struct { /* ... */ }
func (r *Rocket) Launch() { /* ...*/ }

r := new(Rocket)
time.AfterFunc(10 * time.Second, func() {})
```
