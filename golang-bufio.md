#### bufio使用以及源码分析
##### 源码分析

1. type Reader

```
// 实现io.Reader接口，增加buf缓冲
type Reader struct {
	buf          []byte
	rd           io.Reader 
	r, w         int       // 缓冲的读写位置
	err          error
	lastByte     int
	lastRuneSize int
}
```
初始化Reader

```
bufio.NewReader(os.Stdin)
bufio.NewReaderSize(strings.NewReader("golang standard lib"), 1024)
```

对上面初始化方法进行分析:

```
const (
	defaultBufSize = 4096
)
const minReadBufferSize = 16


func NewReader(rd io.Reader) *Reader {
	return NewReaderSize(rd, defaultBufSize)
}

func NewReaderSize(rd io.Reader, size int) *Reader {
	// rd是否已经是*bufio.Reader的对象?
	b, ok := rd.(*Reader)
	// 如果是再比较缓冲大小与传进来的size
	if ok && len(b.buf) >= size {
		return b
	}
	if size < minReadBufferSize {
		size = minReadBufferSize
	}
	// 如果不是，则new一个对象(new返回的是指针类型)
	r := new(Reader)
	// 复位r对象的所有属性值，稍后分析reset方法
	r.reset(make([]byte, size), rd)
	return r
}
```
默认缓冲区大小4096bytes(4kb)，最小不低于16bytes,buf大小永远大于或等于size

2. func (*Reader) Reset

```
// Reset将会放弃缓冲区数据, 重置所有状态并且将b切换到r读取数据
func (b *Reader) Reset(r io.Reader) {
	b.reset(b.buf, r)
}
func (b *Reader) reset(buf []byte, r io.Reader) {
	*b = Reader{
		buf:          buf,
		rd:           r,
		lastByte:     -1,
		lastRuneSize: -1,
	}
}
```

3. func (*Reader) Buffered
Buffered返回缓冲中现有的可读取的字节数

```
func (b *Reader) Buffered() int { 
	//写入缓冲的位置减去已读的位置
	return b.w - b.r
}
```

4. func (*Reader) Peek
回输入流的下n个字节，但不会移动读取位置
先看下fill方法

```
const maxConsecutiveEmptyReads = 100
// 填充读取位置以后的数据到缓冲区的起始位置
func (b *Reader) fill() {
	// 从当前读取的位置开始
	if b.r > 0 {
		// 覆盖已读取数据,b.r之前的数据
		copy(b.buf, b.buf[b.r:b.w])
		//重新设置读写位置
		b.w -= b.r
		b.r = 0
	}
	if b.w >= len(b.buf) {
		panic("bufio: tried to fill full buffer")
	}
	// 读取新数据，如果没有读到数据且没有出错则重试100次
	for i := maxConsecutiveEmptyReads; i > 0; i-- {
		n, err := b.rd.Read(b.buf[b.w:])
		if n < 0 {
			panic(errNegativeRead)
		}
		b.w += n
		if err != nil {
			b.err = err
			return
		}
		if n > 0 {
			return
		}
	}
	b.err = io.ErrNoProgress
}

var (
	ErrInvalidUnreadByte = errors.New("bufio: invalid use of UnreadByte")
	ErrInvalidUnreadRune = errors.New("bufio: invalid use of UnreadRune")
	ErrBufferFull        = errors.New("bufio: buffer full")
	ErrNegativeCount     = errors.New("bufio: negative count")
)

func (b *Reader) readErr() error {
	err := b.err
	b.err = nil
	return err
}
// 整个方法没有移动读写位置，除了fill的时候，可能会有位置移动
func (b *Reader) Peek(n int) ([]byte, error) {
	if n < 0 {
		return nil, ErrNegativeCount
	}
	if n > len(b.buf) {
		return nil, ErrBufferFull
	}
	// 0 <= n <= len(b.buf)
	for b.w-b.r < n && b.err == nil {
		b.fill() // b.w-b.r < len(b.buf) => buffer is not full
	}
	var err error
	if avail := b.w - b.r; avail < n {
		// 缓冲中没有足够的数据
		n = avail
		err = b.readErr()
		if err == nil {
			err = ErrBufferFull
		}
	}
	return b.buf[b.r : b.r+n], err
}
```
5. func (*Reader) Read

```
// 实现自io.Reader的Read方法
func (b *Reader) Read(p []byte) (n int, err error) {
	n = len(p)
	//没有数据
	if n == 0 {
		return 0, b.readErr()
	}
	// 缓冲读写位置一样
	if b.r == b.w {
		if b.err != nil {
			return 0, b.readErr()
		}
		// 字节切片的长度大于等于缓冲区,避免复制，直接读取
		if len(p) >= len(b.buf) {
			n, b.err = b.rd.Read(p)
			if n < 0 {
				panic(errNegativeRead)
			}
			if n > 0 {
				b.lastByte = int(p[n-1])
				b.lastRuneSize = -1
			}
			return n, b.readErr()
		}
		// 缓冲已空
		b.fill()
		if b.r == b.w {
			return 0, b.readErr()
		}
	｝
	// 使用copy方法将buf的数据给p
	n = copy(p, b.buf[b.r:b.w])
	// 设置已读字节数，位置
	b.r += n
	// 最后字读取的节位置
	b.lastByte = int(b.buf[b.r-1])
	// unicode字符大小
	b.lastRuneSize = -1
	return n, nil
}
```

6. func (*Reader) ReadByte 
	读取并返回一个字节。如果没有可用的数据，会返回错误
	
7. func (*Reader) UnreadByte 
	吐出最近一次读取操作读取的最后一个字节，出最近一次读取操作读取的最后一个字节

8. func (*Reader) ReadRune
	ReadRune读取一个utf-8编码的unicode码值，返回该码值、其编码长度和可能的错误

9. func (*Reader) UnreadRune
	UnreadRune吐出最近一次ReadRune调用读取的unicode码值

10. func (*Reader) ReadLine
11. func (*Reader) ReadSlice
12. func (*Reader) ReadBytes
13. func (*Reader) ReadString
14. func (*Reader) WriteTo
	WriteTo方法实现了io.WriterTo接口

15. type Writer
	
```
type Writer struct {
	err error
	buf []byte
	n   int
	wr  io.Writer
}
```

16. type ReadWriter

```
type ReadWriter struct {
    *Reader
    *Writer
}
```

##### bufio的使用
scan方法

```
buf := make([]byte, 20)
index1, buf, e1 := bufio.ScanBytes([]byte("hello, golang"), false)
fmt.Println(index1, string(buf), e1)
index2, buf, e2 := bufio.ScanRunes([]byte("你好，go语言"), false)
fmt.Println(index2, string(buf), e2)
index3, buf, e3 := bufio.ScanWords([]byte("it's a test"), false)
fmt.Println(index3, string(buf), e3)
index4, buf, e4 := bufio.ScanLines([]byte("hi,golang\nit's wonderful"), false)
fmt.Println(index4, string(buf), e4)
```

输出：  
1 h <nil>  
3 你 <nil>  
5 it's <nil>  
10 hi,golang <nil>   

Scanner使用

```
package main

import (
	"bufio"
	"fmt"
	// "os"
	"strings"
	"unicode/utf8"
)

func main() {

	// Scanner
	s1 := bufio.NewScanner(strings.NewReader("let's study golang! she's amazing!"))
	s1.Scan()
	fmt.Println(string(s1.Bytes()))
	fmt.Println(s1.Text())
	// buf := make([]byte, 0, 60)
	// s.Buffer(buf, 10)

	// 一行一行读取
	s2 := bufio.NewScanner(strings.NewReader("let's study golang!\nshe's amazing!"))
	for s2.Scan() {
		fmt.Println(string(s2.Bytes()))
	}
	if err := s2.Err(); err != nil {
		panic(err)
	}

	// 一个词一个词读取
	s3 := bufio.NewScanner(strings.NewReader("let's study golang!\nshe's amazing!"))
	s3.Split(bufio.ScanWords)
	for s3.Scan() {
		fmt.Println(string(s3.Bytes()))
	}
	if err := s3.Err(); err != nil {
		panic(err)
	}

	// 自定义
	s4 := bufio.NewScanner(strings.NewReader("let's study golang!\nshe's amazing!"))
	s4.Split(func(data []byte, atEOF bool) (advance int, token []byte, err error) {
		start := 0
		for width := 0; start < len(data); start += width {
			var r rune
			r, width = utf8.DecodeRune(data[start:])
			if r != int32('s') {
				break
			}
		}
		// Scan until space, marking end of word.
		for width, i := 0, start; i < len(data); i += width {
			var r rune
			r, width = utf8.DecodeRune(data[i:])
			if r == int32('s') {
				return i + width, data[start:i], nil
			}
		}
		// If we're at EOF, we have a final, non-empty, non-terminated word. Return it.
		if atEOF && len(data) > start {
			return len(data), data[start:], nil
		}
		// Request more data.
		return start, nil, nil
	})
	for s4.Scan() {
		fmt.Println(string(s4.Bytes()))
	}
	if err := s4.Err(); err != nil {
		panic(err)
	}

}
```

ReaderWriter的使用

```
package main

import (
	"bufio"
	"fmt"
	"io"
	"os"
	"strings"
)

func main() {
	rw := bufio.NewReadWriter(
		bufio.NewReader(strings.NewReader("i'm studying golang!\n i feel good\ngolang is amazing")),
		bufio.NewWriter(os.Stdout))
	// 逐行读取
	for {
		line, err := rw.ReadString('\n')
		fmt.Println(line)
		if err != nil {
			if err == io.EOF {
				break
			} else {
				panic(err)
			}
		}
	}
	buf := make([]byte, 0, 64)
	n, e := rw.Read(buf)
	if n <= 0 && e != nil {
		panic(e)
	}
	rw.Write(buf)
	rw.Flush()
	rw.WriteString("hi golang")
	rw.WriteTo(rw)
	rw.Flush()
	fmt.Println(rw.ReadString('\n'))
}
```




