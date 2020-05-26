# 聊聊Golang中的字符串



## 引言





笔者最近开发了一个服务，功能是缓存股票k线数据，减少redis的压力。每一根k线对应redis里的一个用`|`分隔的字符串，如`"2020-02-19|186.72|188.06|188.18|186.47|187.28|29997471|5618676305.65|0|0.39|0.80|"`。从redis里拿到字符串之后，对其进行分割，然后转换成结构体，放在内存里作为缓存。rpc接口直接拿内存里的数据返回就可以了。



在测试的过程中，发现了一个奇怪的现象，那就是通过pprof查看内存占用时， 发现除了内存中的结构体区域之外，还有一块大小相近的字符串内存区域。

为了彻底搞清楚以上现象的原因，笔者认为有必要深入了解一下golang中的字符串，因此有了本文。



## 数据结构



先来看一下string在运行时的结构，其定义在文件`src/reflect/value.go`中



```

type StringHeader struct {

    Data uintptr

    Len int

}

```



对比一下slice的结构，可以发现它们非常相似。



```

type SliceHeader struct {

    Data uintptr

    Len int

    Cap int

}

```



string比slice少了一个`Cap`字段，因为string是只读的。



`Data`字段是个指针，指向底层的字节序列。

当对字符串类型使用赋值操作时，目标字符串和源字符串会共享底层的字节序列。

```
s1 := "hello world"
s2 := s1
s3 := s1[:5]
x1 := (*reflect.StringHeader)(unsafe.Pointer(&s1))
x2 := (*reflect.StringHeader)(unsafe.Pointer(&s2))
x3 := (*reflect.StringHeader)(unsafe.Pointer(&s3))
```

上面的代码中，`&s1`和`&s2`是不相等的，但是`x1.Data`和`x2.Data`是相等的。当我们使用语法`aString[start:end]`来获取子串的时候，也会共享底层的字节序列，因此`x1.Data`和`x3.Data`是相等的。


## string、unicode、utf8、rune



要深入了解golang中的字符串，绕不开这几个概念。下面依次介绍一下。



**unicode**是一套业界标准，它包括「字符集」和「编码方案」等内容。它为每一个「字符」定义唯一的代码，这个唯一的代码叫做**code point**。

下面来举例。英文字母`a`的code point是`U+0061`，中文`好`的code point是`U+597D`。unicode还支持组合字符(combining character)，通过多个code point来组成一个组合字符，比如`é`可以由`U+0065`(e)和`U+0301`(´)这两个code point组合来表示。同时，为了兼容性，unicode里还有一些预组合字符(precomposed character)，比如`é`有单独的code point`U+00E9`。



unicode定义的code point的范围从`U+0000`到`U+10FFFF`，对应的编码方案有多种，包括UTF-8、UTF-16、UTF-32等。UTF-8使用1~4个字节来编码一个code point。

下面来举例。英文字母`a`的code point是`U+0061`，UTF-8编码为`0x61`。中文`好`的code point是`U+597D`，UTF-8编码为`0xE5A5BD`。



下面来聊聊golang中的string、byte、string literal、rune这几个概念。

**byte**对于我们开发人员来说是最熟悉不过的了，1 byte=8 bits。golang中的byte是uint8的别名。

**string**是只读的slice of bytes，可以包含任意的字节，并不要求必须是UTF-8格式的。

**string literal**中文名是字符串字面量。golang中的string literal是那些直接写在源代码中的字符串，也就是源代码中通过双引号或者反引号定义的字符串。因为golang的源代码都是UTF-8的，所以golang中的字符串字面量基本上都是UTF-8的，比如`"abc"`这种常规的字符串字面量。像`"\xbd\xb2"`这种使用`\xNN`格式、故意写成非UTF8编码的字面量，也是合法的字符串字面量。

**rune**是golang中对code point的简称，也是int32的别名。



## 类型转换



string和[]byte之间能够进行显式的类型转换。

string和[]rune之间也能够进行显式的类型转换。



来看下面这个示例程序



```

package main

import "fmt"

func main() {

    s := "Hello, 世界"

    fmt.Printf("%+v\n", s)

    fmt.Printf("%+v\n", []byte(s))

    fmt.Printf("%+v\n", string([]byte(s)))

    fmt.Printf("%+v\n", []rune(s))

    fmt.Printf("%+v\n", string([]rune(s)))

}

```

它的结果如下

```

Hello, 世界

[72 101 108 108 111 44 32 228 184 150 231 149 140]

Hello, 世界

[72 101 108 108 111 44 32 19990 30028]

Hello, 世界

```



需要注意的是，以上的类型转换都会发生内存拷贝。因此在高并发场景下，使用这些强制类型转换，性能上会比较差。

- `[]byte->string`调用了`runtime.slicebytetostring`

- `string->[]byte`调用了`runtime.stringtoslicebyte`

- `[]rune->string`调用了`runtime.slicerunetostring`

- `string->[]rune`调用了`runtime.stringtoslicerune`



有兴趣的同学可以去看看源码，一般来说我们知道这种转换会发生内存拷贝就够了。

另外，在某些场景下，编译器对string和byte slice之间的转换做了优化，不会发生拷贝，比如在for-range loop中将string转换成byte slice(`for i, b := range []byte(s) {}`)。

## 拼接



golang中拼接字符串的方式有很多种，相关的文章也很多。各种拼接方式的性能会受待拼接字符串的长度和数量影响，要全面测试比较困难。这里主要介绍各种拼接方式的实现原理，因此只针对拼接10个、10000个短字符串进行了性能测试。

benchmark测试代码在这个[gist](https://gist.github.com/yrc0d3/6964e81b861efc23754cba5a38859e1b)中。笔者在自己的笔记本（2.5 GHz Intel Core i7）上测试了一下，结果如下：

```
BenchmarkJoin10-4                                9577982               125 ns/op             112 B/op          1 allocs/op
BenchmarkSprintf10-4                             4068709               297 ns/op             112 B/op          1 allocs/op
BenchmarkConcat10-4                              2534871               482 ns/op             608 B/op          9 allocs/op
BenchmarkBytesBuffer10-4                         4680453               255 ns/op             320 B/op          3 allocs/op
BenchmarkBytesBufferWithGrow10-4                 5702677               218 ns/op             224 B/op          2 allocs/op
BenchmarkStringBuilder10-4                       6260330               193 ns/op             240 B/op          4 allocs/op
BenchmarkStringBuilderWithGrow10-4              13437114              86.7 ns/op             112 B/op          1 allocs/op
BenchmarkJoin10000-4                               10000            109380 ns/op          106693 B/op          1 allocs/op
BenchmarkSprintf10000-4                             3282            335898 ns/op          629595 B/op         25 allocs/op
BenchmarkConcat10000-4                                16          68492777 ns/op        531099348 B/op     10015 allocs/op
BenchmarkBytesBuffer10000-4                         8527            143114 ns/op          423706 B/op         13 allocs/op
BenchmarkBytesBufferWithGrow10000-4                10000            101874 ns/op          213143 B/op          2 allocs/op
BenchmarkStringBuilder10000-4                       8168            133852 ns/op          522409 B/op         23 allocs/op
BenchmarkStringBuilderWithGrow10000-4              19359             60950 ns/op          106591 B/op          1 allocs/op
```

（PS：如果选取的测试字符串太短的话，有可能会被golang优化到没有内存分配。上面的测试选择的字符串长度适中，能够避开golang对短字符串拼接的优化。）

从中我们可以看出，Join、BytesBufferWithGrow和StringBuilderWithGrow是性能最好的，耗时短且内存分配少。下面逐个分析一下。


### +

拼接字符串最简单的方式就是使用`+`。这也是我们的测试中性能最差的方式。

通过`+`来进行字符串拼接，编译器会将原始代码转换成通过函数`cmd/compile/internal/gc.addstr`来生成拼接字符串的代码。这个函数帮助我们在编译期间选择合适的函数来拼接字符串，但最终都会调用函数`runtime.concatstrings`。这个函数中，一般都会先分配一块能够容纳所有待拼接字符串的内存空间，然后通过调用`copy`函数来组装最终的字符串。

那么为什么这种拼接方式在测试中的表现最差呢？这是因为编译器转换代码是按语句来操作的。我们在for循环来使用`+`拼接所有字符串，因此对长度为n的字符串数组来说，要进行n次内存分配。因此性能差。

在已知待拼接字符串数量的时候，将其代码写在一行，如`s := s1 + s2 + s3`，其性能是非常高的，因为只有一次内存分配。在这种场景下，语法简单，性能又好，推荐使用。

### fmt.Sprintf

其源码如下：

```
func Sprintf(format string, a ...interface{}) string {
	p := newPrinter()
	p.doPrintf(format, a)
	s := string(p.buf)
	p.free()
	return s
}
```

`fmt.Sprintf`的内部实现中，通过`newPrinter`函数创建了工具类`fmt.pp`(维护打印状态)，通过其`doPrintf`函数来完成字符串的拼接。`p.doPrintf`函数中的主要原理就是遍历`format`字符串，找到其中的`%`，然后根据后面的动词将对应的参数变量转换成字符串，然后继续遍历直到结束。

`p.doPrintf`函数内部最终是通过`p.buf.writeString`来拼接字符串。其代码如下：

```
// Use simple []byte instead of bytes.Buffer to avoid large dependency.
type buffer []byte

func (b *buffer) writeString(s string) {
	*b = append(*b, s...)
}
```

因此，其内存分配次数是取决于slice的grow策略的。

### bytes.Buffer

bytes.Buffer会根据输入的数据长度来动态申请内存。其内部结构很简单，主要就是一个byte slice。其源码如下：

```
type Buffer struct {
	buf      []byte // contents are the bytes buf[off : len(buf)]
	off      int    // read at &buf[off], write at &buf[len(buf)]
	lastRead readOp // last read operation, so that Unread* can work correctly.
}
```

测试中的`BytesBufferWithGrow`和`BytesBuffer`唯一的区别就是`BytesBufferWithGrow`会提前调用`grow`函数分配足够长度的内存。通过其测试结果对比可以看出，此举能够提升性能。

### strings.Builder

`strings.Builder`是go在1.10版本中加入的，也是官方推荐的用于拼接字符串的结构。

它的内部结构非常简单，基本就是一个byte slice。所有的操作都是基于这个byte slice。其源码如下：

```
type Builder struct {
	addr *Builder // of receiver, to detect copies by value
	buf  []byte
}
```

从测试结果中可以看出，虽然底层实现都是byte slice，但在提前使用`grow`函数申请内存的场景下，`strings.Builder`比`bytes.Buffer`要少了一次内存分配，性能也更好一些。原因在于我们最后都调用了它们的`String`方法来获取最终的字符串。`bytes.Buffer`的`String`方法会将byte slice强制转换成string，结合文章前面的内容我们知道，这一步是要进行内存拷贝的。而`strings.Builder`的`string`方法使用了`unsafe.Pointer`来将byte slice转换成string，避免了内存拷贝。

```
func (b *Buffer) String() string {
	if b == nil {
		// Special case, useful in debugging.
		return "<nil>"
	}
	return string(b.buf[b.off:])
}

func (b *Builder) String() string {
	return *(*string)(unsafe.Pointer(&b.buf))
}
```

### strings.Join

其内部实现使用了`strings.Builder`，并在开始就通过`Grow`函数预分配了待拼接字符串长度和的内存，因此性能很好。

### 总结

通过以上分析，我们可以看出，影响字符串拼接性能的主要因素就是内存分配次数。无论使用哪种拼接方式，尽量减少内存分配次数，就能获取更好的性能。

如果有拼接非字符串或者特殊格式化的需求，那么估计只能选择fmt系列函数。如果编译时能够确定待拼接字符串数量，那么在一行内使用`+`将其拼接起来是既方便又高效的。其他场景，使用`strings.Buider`一般都是最好的选择。



## 遍历

golang中遍历字符串，我们一般都会使用for-range loop。需要注意的是，使用for-range来遍历字符串时，我们拿到的值都是rune类型的变量，而不是byte。`for v1, v2 := range a {}`会在编译时转换成如下形式的代码：

```
ha := a
for hv1 := 0; hv1 < len(ha); {
    hv1t := hv1
    hv2 := rune(ha[hv1])
    if hv2 < utf8.RuneSelf {
       hv1++
    } else {
       hv2, hv1 = decoderune(ha, hv1)
    }
    v1, v2 = hv1t, hv2
    // original body
}
```

当我们使用下标的方式来遍历字符串时，像`for i = 0; i < len(s); i++ {}`，`s[i]`是byte类型的变量，这种情况就是把字符串当作是字节数组来使用了。

## 字符串比较

当我们使用`==`或者`!=`来比较两个字符串的时候，如果它们底层的字节序列地址相同，那么可以通过比较其长度来判断其是否相等，时间复杂度是`O(1)`。如果底层的字节序列地址不同，那么时间复杂度就是`O(n)`了。

感兴趣的可以看一下[汇编源码](https://github.com/golang/go/blob/9b189686a53d7fec7deb93d7521531157aa023cb/src/internal/bytealg/compare_amd64.s#L30)

## 使用字符串内部化来优化空间占用

**字符串内部化**(string intern)是指一种让相同的字符串在内存中只保存一份的技术。对于要存储大量字符串的应用来说，它可以显著降低内存占用。

通过前述分析，我们知道golang中的字符串结构体为`StringHeader`。其中的`Data`是一个指向底层字节序列的地址。两个字符串变量在内存中的地址虽然不同，但它们的底层字节序列的地址可以相同。

golang对于在编译期间可以确定的字符串常量，会进行内部化处理。如果是运行期间产生的字符串，则无法内部化。

```
// 可以被intern，底层字节序列地址相同
s1 := "12"
s2 := "1"+"2"

// 不能被intern，底层字节序列地址不同
s3 := "12"
s4 := strconv.Itoa(12)
```

了解这些之后，我们可以想办法绕过限制，在运行时手动完成字符串内部化。一个简单的实现如下：

```
package main

import (
	"fmt"
	"reflect"
	"strconv"
	"unsafe"
)

// stringptr returns a pointer to the string data.
func stringptr(s string) uintptr {
	return (*reflect.StringHeader)(unsafe.Pointer(&s)).Data
}

type stringInterner map[string]string

func (si stringInterner) Intern(s string) string {
	if interned, ok := si[s]; ok {
		return interned
	}
	si[s] = s
	return s
}

func main() {
	si := stringInterner{}
	s1 := si.Intern("12")
	s2 := si.Intern(strconv.Itoa(12))
	fmt.Println(stringptr(s1) == stringptr(s2)) // true
}
```

其原理很简单，就是自己维护一个字符串的池子，在程序中只使用池子中的字符串。在缓存类的应用中，使用该技术可以节省很多内存空间。

golang在net包中也使用了该技术：

```
// commonHeader interns common header strings.
var commonHeader map[string]string

var commonHeaderOnce sync.Once

func initCommonHeader() {
	commonHeader = make(map[string]string)
	for _, v := range []string{
		"Accept",
		"Accept-Charset",
		"Accept-Encoding",
		// ...
	} {
		commonHeader[v] = v
	}
}
```

## 结语

结合以上内容，就可以解决文章开头遇到的问题。在将redis中的k线字符串转换为结构体的时候，结构体引用了字符串split之后的第一个表示日期的子串，因此导致原始字符串的底层的字节序列未被回收，一直驻留在内存中。使用字符串内部化技术，可以很简单的解决问题，同时减少内存占用。

以文章开头的字符串`"2020-02-19|186.72|188.06|188.18|186.47|187.28|29997471|5618676305.65|0|0.39|0.80|"`为例。通过阅读`strings.Split`的源码可知，它是通过设置offset，复用底层字节数组来生成的子字符串。

```
func Split(s, sep string) []string { return genSplit(s, sep, 0, -1) }

func genSplit(s, sep string, sepSave, n int) []string {
	if n == 0 {
		return nil
	}
	if sep == "" {
		return explode(s, n)
	}
	if n < 0 {
		n = Count(s, sep) + 1
	}

	a := make([]string, n)
	n--
	i := 0
	for i < n {
		m := Index(s, sep)
		if m < 0 {
			break
		}
		a[i] = s[:m+sepSave]
		s = s[m+len(sep):]
		i++
	}
	a[i] = s
	return a[:i+1]
}
```

在`strings.Split`之后，我们能够获取一个字符串数组，暂命名其为`kFields`。然后将`kFields`的各个值分别赋给结构体的各个字段。`kFields[0]`是日期，仍然被保留为string类型。其他字段都需要进行到数字的转换。因此，只有`kFields[0]`一直被引用着，也就导致原始字符串的底层字节序列一直被引用，无法被垃圾回收，也就一直占用着内存。通过内部化技术解除对`kFields[0]`的引用之后，原始字符串的底层字节序列也可以被回收掉了，内存占用也就降下来了。

以上就是笔者通过实际问题，逐渐深入内部原理，然后对golang中字符串的相关内容进行的梳理。如有错误之处，敬请指正。


## 参考资料



- [Strings, bytes, runes and characters in Go](https://blog.golang.org/strings)

- [Basic Types and Basic Value Literals](https://go101.org/article/basic-types-and-value-literals.html)

- [Strings in Go](https://go101.org/article/string.html)

- [String interning in Go](https://artem.krylysov.com/blog/2018/12/12/string-interning-in-go/)

- [聊一聊字符串内部化](https://ms2008.github.io/2019/08/18/golang-string-interning/)

- [【Go】string 优化误区及建议](https://blog.thinkeridea.com/201902/go/string_ye_shi_yin_yong_lei_xing.html)

- [Go语言字符串高效拼接](https://www.flysnow.org/2018/10/28/golang-concat-strings-performance-analysis.html)



