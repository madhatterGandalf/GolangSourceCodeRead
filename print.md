golang源码阅读



# 1.fmt包打印字符串数据的源码

```go
package main

import "fmt"

func main() {
	var message string
	message = "Hello World."
	fmt.Print(message)
}
```

## 1.1.调用print.go (总的调用逻辑)

```go
func Print(a ...interface{}) (n int, err error) {
    return Fprint(os.Stdout, a...)		//Fprint调用的方法  (os.Stdout标准的输出流)
}

//pp 结构体
type pp struct {
	buf buffer

	//存放真是打印数据
	arg interface{}

	//打印数据的反射数据
	value reflect.Value

	// fmt is used to format basic items such as integers or strings.
	fmt fmt

	// reordered records whether the format string used argument reordering.
	reordered bool
	// goodArgNum records whether the most recent reordering directive was valid.
	goodArgNum bool
	// panicking is set by catchPanic to avoid infinite panic, recover, panic, ... recursion.
	panicking bool
	// erroring is set when printing an error string to guard against calling handleMethods.
	erroring bool
}

//a 转数组
func Fprint(w io.Writer, a ...interface{}) (n int, err error) {
	p := newPrinter()		//使用线程池  获取一个线程
	p.doPrint(a)						
	n, err = w.Write(p.buf)
	p.free()				//放回线程到线程池
	return
}

```

### 1.1.1 p.doPrint(a) 的操作  (给结构体pp赋值的过程)

```go
func (p *pp) doPrint(a []interface{}) {
	prevString := false
	for argNum, arg := range a {
        //判断参数值是否是string类型
		isString := arg != nil && reflect.TypeOf(arg).Kind() == reflect.String
		// Add a space between two non-string arguments.
		if argNum > 0 && !isString && !prevString {
			p.buf.WriteByte(' ')
		}
        
		p.printArg(arg, 'v')
		prevString = isString
	}
}
```

### 1.1.2 p.printArg(arg, 'v')

```go
//arg = Hello World.	verb = 118  'V'
func (p *pp) printArg(arg interface{}, verb rune) {
	p.arg = arg					//赋值结构体
	p.value = reflect.Value{}	 //赋值结构体

	if arg == nil {
		switch verb {
		case 'T', 'v':
			p.fmt.padString(nilAngleString)
		default:
			p.badVerb(verb)
		}
		return
	}

	// Special processing considerations.
	// %T (the value's type) and %p (its address) are special; we always do them first.
	switch verb {
	case 'T':
		p.fmt.fmt_s(reflect.TypeOf(arg).String())
		return
	case 'p':
		p.fmtPointer(reflect.ValueOf(arg), 'p')
		return
	}

	// Some types can be done without reflection.
    //判断字符串的类型
	switch f := arg.(type) {
	case bool:
		p.fmtBool(f, verb)
	case float32:
		p.fmtFloat(float64(f), 32, verb)
	case float64:
		p.fmtFloat(f, 64, verb)
	case complex64:
		p.fmtComplex(complex128(f), 64, verb)
	case complex128:
		p.fmtComplex(f, 128, verb)
	case int:
		p.fmtInteger(uint64(f), signed, verb)
	case int8:
		p.fmtInteger(uint64(f), signed, verb)
	case int16:
		p.fmtInteger(uint64(f), signed, verb)
	case int32:
		p.fmtInteger(uint64(f), signed, verb)
	case int64:
		p.fmtInteger(uint64(f), signed, verb)
	case uint:
		p.fmtInteger(uint64(f), unsigned, verb)
	case uint8:
		p.fmtInteger(uint64(f), unsigned, verb)
	case uint16:
		p.fmtInteger(uint64(f), unsigned, verb)
	case uint32:
		p.fmtInteger(uint64(f), unsigned, verb)
	case uint64:
		p.fmtInteger(f, unsigned, verb)
	case uintptr:
		p.fmtInteger(uint64(f), unsigned, verb)
	case string:	//-------------------------------赋值pp结构体 fmt 的值
		p.fmtString(f, verb)
	case []byte:
		p.fmtBytes(f, verb, "[]byte")
	case reflect.Value:
		// Handle extractable values with special methods
		// since printValue does not handle them at depth 0.
		if f.IsValid() && f.CanInterface() {
			p.arg = f.Interface()
			if p.handleMethods(verb) {
				return
			}
		}
		p.printValue(f, verb, 0)
	default:
		// If the type is not simple, it might have methods.
		if !p.handleMethods(verb) {
			// Need to use reflection, since the type had no
			// interface methods that could be used for formatting.
			p.printValue(reflect.ValueOf(f), verb, 0)
		}
	}
}
```

### 1.1.3.p.fmtString(f, verb)

```go
func (p *pp) fmtString(v string, verb rune) { verb == 118
	switch verb {
	case 'v':	'v' = 118
		if p.fmt.sharpV {		// 取默认值是 false
			p.fmt.fmt_q(v)
		} else {
			p.fmt.fmt_s(v)		//---
		}
	case 's':
		p.fmt.fmt_s(v)
	case 'x':
		p.fmt.fmt_sx(v, ldigits)
	case 'X':
		p.fmt.fmt_sx(v, udigits)
	case 'q':
		p.fmt.fmt_q(v)
	default:
		p.badVerb(verb)
	}
}
```

### 1.1.4.调用format包中的fmt_s方法

```go
func (f *fmt) fmt_s(s string) {
   s = f.truncate(s)	//返回值没有变化
   f.padString(s)		//---
}
```

### 1.1.5. f.padString(s)

```go
func (f *fmt) padString(s string) {
   if !f.widPresent || f.wid == 0 {	//判断是否是初始化的值
      f.buf.WriteString(s)			//给buffer赋值
      return
   }
   width := f.wid - utf8.RuneCountInString(s)
   if !f.minus {
      // left padding
      f.writePadding(width)
      f.buf.WriteString(s)
   } else {
      // right padding
      f.buf.WriteString(s)
      f.writePadding(width)
   }
}
```

1.1.6.  buffer的本质

```go
type buffer []byte			//底层数据结构

func (b *buffer) Write(p []byte) {
   *b = append(*b, p...)
}

func (b *buffer) WriteString(s string) {
   *b = append(*b, s...)
}

//  结构体数据 赋值完成
```

# 1.2 赋值完成后  n, err = w.Write(p.buf) 标准的输出流

## 1.2.1  os.Stdout 的值  (不做深入解析)

```go
func NewFile(fd uintptr, name string) *File {		//*File 类型
   h := syscall.Handle(fd)
   if h == syscall.InvalidHandle {
      return nil
   }
   return newFile(h, name, "file")
}
```

```go
func newFile(h syscall.Handle, name string, kind string) *File {
   if kind == "file" {
      var m uint32
      if syscall.GetConsoleMode(h, &m) == nil {
         kind = "console"
      }
   }

   f := &File{&file{
      pfd: poll.FD{
         Sysfd:         h,
         IsStream:      true,
         ZeroReadIsEOF: true,
      },
      name: name,
   }}
   runtime.SetFinalizer(f.file, (*file).close)

   // Ignore initialization errors.
   // Assume any problems will show up in later I/O.
   f.pfd.Init(kind, false)

   return f
}
```

## 1.2.2  线程池分析

```go
func Fprint(w io.Writer, a ...interface{}) (n int, err error) {
   p := newPrinter()		//获取线程
   p.doPrint(a)
   n, err = w.Write(p.buf)
   p.free()					//释放
   return
}
```

## 1.2.3 p := newPrinter() 获取分析

```go
var ppFree = sync.Pool{		//初始化pool
   New: func() interface{} { return new(pp) },
}

// newPrinter allocates a new pp struct or grabs a cached one.
func newPrinter() *pp {
   p := ppFree.Get().(*pp)
   p.panicking = false
   p.erroring = false
   p.fmt.init(&p.buf)
   return p
}
```

pool 结构体

```go
type Pool struct {
   noCopy noCopy

   local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
   localSize uintptr        // size of the local array

   // New optionally specifies a function to generate
   // a value when Get would otherwise return nil.
   // It may not be changed concurrently with calls to Get.
   New func() interface{}
}
```

//放回到线程池

```go
func (p *pp) free() {
   p.buf = p.buf[:0]
   p.arg = nil
   p.value = reflect.Value{}
   ppFree.Put(p)
}
```

