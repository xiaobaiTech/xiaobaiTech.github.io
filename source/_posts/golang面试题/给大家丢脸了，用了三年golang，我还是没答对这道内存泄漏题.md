---
title: 给大家丢脸了，用了三年golang，我还是没答对这道内存泄漏题。
date: 2020-11-12 22:57:55
tags:
categories: "golang面试题"
---

[![](https://s3.ax1x.com/2020/11/20/DMGVKJ.png)](https://imgchr.com/i/DMGVKJ)

<!-- more -->
## 问题

```go
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
	"runtime"
)

func main() {
	num := 6
	for index := 0; index < num; index++ {
		resp, _ := http.Get("https://www.baidu.com")
		_, _ = ioutil.ReadAll(resp.Body)
	}
	fmt.Printf("此时goroutine个数= %d\n", runtime.NumGoroutine())
}

```

**上面这道题在不执行`resp.Body.Close()`的情况下，泄漏了吗？如果泄漏，泄漏了多少个`goroutine`?**
	

## 怎么答
- **不进行`resp.Body.Close()`，泄漏是一定的**。但是泄漏的`goroutine`个数就让我迷糊了。由于执行了**6遍**，每次泄漏一个**读和写goroutine**，就是**12个goroutine**，加上`main函数`本身也是一个`goroutine`，所以答案是**13**.
- 然而执行程序，发现**答案是3**，出入有点大，为什么呢？

## 解释
- 我们直接看源码。`golang` 的 `http` 包。
  
```go
http.Get()

-- DefaultClient.Get
----func (c *Client) do(req *Request)
------func send(ireq *Request, rt RoundTripper, deadline time.Time)
-------- resp, didTimeout, err = send(req, c.transport(), deadline) 
// 以上代码在 go/1.12.7/libexec/src/net/http/client:174 

func (c *Client) transport() RoundTripper {
	if c.Transport != nil {
		return c.Transport
	}
	return DefaultTransport
}
```

- 说明 `http.Get` 默认使用 `DefaultTransport` 管理连接。
#####   `DefaultTransport` 是干嘛的呢？

```go
// It establishes network connections as needed
// and caches them for reuse by subsequent calls.
```
- `DefaultTransport` 的作用是根据需要建立网络连接并缓存它们以供后续调用重用。
#####  那么 `DefaultTransport` 什么时候会建立连接呢？
接着上面的代码堆栈往下翻

```go
func send(ireq *Request, rt RoundTripper, deadline time.Time) 
--resp, err = rt.RoundTrip(req) // 以上代码在 go/1.12.7/libexec/src/net/http/client:250
func (t *Transport) RoundTrip(req *http.Request)
func (t *Transport) roundTrip(req *Request)
func (t *Transport) getConn(treq *transportRequest, cm connectMethod)
func (t *Transport) dialConn(ctx context.Context, cm connectMethod) (*persistConn, error) {
    ...
	go pconn.readLoop()  // 启动一个读goroutine
	go pconn.writeLoop() // 启动一个写goroutine
	return pconn, nil
}
```

- 一次建立连接，就会启动一个`读goroutine`和`写goroutine`。这就是为什么一次`http.Get()`会泄漏`两个goroutine`的来源。
- 泄漏的来源知道了，也知道是因为没有执行`close`

#####  那为什么不执行 `close` 会泄漏呢？
- 回到刚刚启动的`读goroutine` 的 `readLoop()` 代码里 
  
```go
func (pc *persistConn) readLoop() {
	alive := true
	for alive {
        ...
		// Before looping back to the top of this function and peeking on
		// the bufio.Reader, wait for the caller goroutine to finish
		// reading the response body. (or for cancelation or death)
		select {
		case bodyEOF := <-waitForBodyRead:
			pc.t.setReqCanceler(rc.req, nil) // before pc might return to idle pool
			alive = alive &&
				bodyEOF &&
				!pc.sawEOF &&
				pc.wroteRequest() &&
				tryPutIdleConn(trace)
			if bodyEOF {
				eofc <- struct{}{}
			}
		case <-rc.req.Cancel:
			alive = false
			pc.t.CancelRequest(rc.req)
		case <-rc.req.Context().Done():
			alive = false
			pc.t.cancelRequest(rc.req, rc.req.Context().Err())
		case <-pc.closech:
			alive = false
        }
        ...
	}
}

```

- 简单来说`readLoop`就是一个死循环，只要`alive`为`true`，`goroutine`就会一直存在
- `select` 里是 `goroutine` **有可能**退出的场景：
  - `body` 被读取完毕或`body`关闭
  - `request` 主动 `cancel`
  - `request` 的 `context Done` 状态 `true`
  - 当前的 `persistConn` 关闭
  
其中第一个 `body` 被读取完或关闭这个 `case`:

```go
alive = alive &&
    bodyEOF &&
    !pc.sawEOF &&
    pc.wroteRequest() &&
    tryPutIdleConn(trace)
```

`bodyEOF` 来源于到一个通道 `waitForBodyRead`，这个字段的 `true` 和 `false` 直接决定了 `alive` 变量的值（`alive=true`那`读goroutine`继续活着，循环，否则退出`goroutine`）。

##### 那么这个通道的值是从哪里过来的呢？


```go
// go/1.12.7/libexec/src/net/http/transport.go: 1758
		body := &bodyEOFSignal{
			body: resp.Body,
			earlyCloseFn: func() error {
				waitForBodyRead <- false
				<-eofc // will be closed by deferred call at the end of the function
				return nil

			},
			fn: func(err error) error {
				isEOF := err == io.EOF
				waitForBodyRead <- isEOF
				if isEOF {
					<-eofc // see comment above eofc declaration
				} else if err != nil {
					if cerr := pc.canceled(); cerr != nil {
						return cerr
					}
				}
				return err
			},
		}
```

- 如果执行 `earlyCloseFn` ，`waitForBodyRead` 通道输入的是 `false`，`alive` 也会是 `false`，那 `readLoop()` 这个 `goroutine` 就会退出。
- 如果执行 `fn` ，其中包括正常情况下 `body` 读完数据抛出 `io.EOF` 时的 `case`，`waitForBodyRead` 通道输入的是 `true`，那 `alive` 会是 `true`，那么 `readLoop()` 这个 `goroutine` 就不会退出，同时还顺便执行了 `tryPutIdleConn(trace)` 。
  

```go
// tryPutIdleConn adds pconn to the list of idle persistent connections awaiting
// a new request.
// If pconn is no longer needed or not in a good state, tryPutIdleConn returns
// an error explaining why it wasn't registered.
// tryPutIdleConn does not close pconn. Use putOrCloseIdleConn instead for that.
func (t *Transport) tryPutIdleConn(pconn *persistConn) error
```

- `tryPutIdleConn` 将 `pconn` 添加到等待新请求的空闲持久连接列表中，也就是之前说的连接会复用。

#####  那么问题又来了，什么时候会执行这个 `fn` 和 `earlyCloseFn` 呢？

```go
func (es *bodyEOFSignal) Close() error {
	es.mu.Lock()
	defer es.mu.Unlock()
	if es.closed {
		return nil
	}
	es.closed = true
	if es.earlyCloseFn != nil && es.rerr != io.EOF {
		return es.earlyCloseFn() // 关闭时执行 earlyCloseFn
	}
	err := es.body.Close()
	return es.condfn(err)
}
```

- 上面这个其实就是我们比较收悉的 `resp.Body.Close()` ,在里面会执行 `earlyCloseFn`，也就是此时 `readLoop()` 里的 `waitForBodyRead` 通道输入的是 `false`，`alive` 也会是 `false`，那 `readLoop()` 这个 `goroutine` 就会退出，`goroutine` 不会泄露。
  
```go
b, err = ioutil.ReadAll(resp.Body)
--func ReadAll(r io.Reader) 
----func readAll(r io.Reader, capacity int64) 
------func (b *Buffer) ReadFrom(r io.Reader)


// go/1.12.7/libexec/src/bytes/buffer.go:207
func (b *Buffer) ReadFrom(r io.Reader) (n int64, err error) {
	for {
		...
		m, e := r.Read(b.buf[i:cap(b.buf)])  // 看这里，是body在执行read方法
		...
	}
}
```

- 这个`read`，其实就是 `bodyEOFSignal` 里的
  
```go
func (es *bodyEOFSignal) Read(p []byte) (n int, err error) {
	...
	n, err = es.body.Read(p)
	if err != nil {
		... 
    // 这里会有一个io.EOF的报错，意思是读完了
		err = es.condfn(err)
	}
	return
}


func (es *bodyEOFSignal) condfn(err error) error {
	if es.fn == nil {
		return err
	}
	err = es.fn(err)  // 这了执行了 fn
	es.fn = nil
	return err
}
```

- 上面这个其实就是我们比较收悉的读取 `body` 里的内容。 `ioutil.ReadAll()` ,在读完 `body` 的内容时会执行 `fn`，也就是此时 `readLoop()` 里的 `waitForBodyRead` 通道输入的是 `true`，`alive` 也会是 `true`，那 `readLoop()` 这个 `goroutine` 就不会退出，`goroutine` 会泄露，然后执行 `tryPutIdleConn(trace)` 把连接放回池子里复用。
##  总结
- 所以结论呼之欲出了，虽然执行了 `6` 次循环，而且每次都没有执行 `Body.Close()` ,就是因为执行了`ioutil.ReadAll()`把内容都读出来了，连接得以复用，因此只泄漏了一个`读goroutine`和一个`写goroutine`，最后加上`main goroutine`，所以答案就是`3个goroutine`。
- 从另外一个角度说，正常情况下我们的代码都会执行 `ioutil.ReadAll()`，但如果此时忘了 `resp.Body.Close()`，确实会导致泄漏。但如果你**调用的域名一直是同一个**的话，那么只会泄漏一个 `读goroutine` 和一个`写goroutine`，**这就是为什么代码明明不规范但却看不到明显内存泄漏的原因**。
- 那么问题又来了，为什么上面要特意强调是同一个域名呢？改天，回头，以后有空再说吧。



## 文章推荐：
- [连nil切片和空切片一不一样都不清楚？那BAT面试官只好让你回去等通知了。](https://mp.weixin.qq.com/s/myGJ4TrEoVGqLAN3tbZHMw) 
- [昨天那个在for循环里append元素的同事，今天还在么？](https://mp.weixin.qq.com/s/SHxcspmiKyPwPBbhfVxsGA) 
- [golang面试官：for select时，如果通道已经关闭会怎么样？如果只有一个case呢？](https://mp.weixin.qq.com/s/lK6I353Iw08robqpmPB6-g) 
- [golang面试官：for select时，如果通道已经关闭会怎么样？如果只有一个case呢？](https://mp.weixin.qq.com/s/lK6I353Iw08robqpmPB6-g) 
- [golang面试题：对已经关闭的的chan进行读写，会怎么样？为什么？](https://mp.weixin.qq.com/s/qm-8pvHBVRmLQQ4_DHc1Tw) 
- [golang面试题：对未初始化的的chan进行读写，会怎么样？为什么？](https://mp.weixin.qq.com/s/ixJu0wrGXsCcGzveCqnr6A) 
- [golang 面试题：reflect（反射包）如何获取字段 tag？为什么 json 包不能导出私有变量的 tag？](https://mp.weixin.qq.com/s/WK9StkC3Jfy-o1dUqlo7Dg) 
- [golang面试题：json包变量不加tag会怎么样？](https://mp.weixin.qq.com/s/zZM_iLuopyenI0LD6VYZGw) 
- [golang面试题：怎么避免内存逃逸？](https://mp.weixin.qq.com/s/4QAxGEr9KxtZXyfSG8VoCQ) 
- [golang面试题：简单聊聊内存逃逸？](https://mp.weixin.qq.com/s/4YYR1eYFIFsNOaTxL4Q-eQ) 
- [golang面试题：字符串转成byte数组，会发生内存拷贝吗？](https://mp.weixin.qq.com/s/d80m0hgoKcHfKp4ZXH1M4A)  
- [golang面试题：翻转含有中文、数字、英文字母的字符串](https://mp.weixin.qq.com/s/OIRPOszH-rTJp03AeRgnRQ)  
- [golang面试题：拷贝大切片一定比小切片代价大吗？](https://mp.weixin.qq.com/s/hPYdiHYRufimyKT4FcW4HA)   
- [golang面试题：能说说uintptr和unsafe.Pointer的区别吗？](https://mp.weixin.qq.com/s/IkOwh9bh36vK6JgN7b3KjA)

###### 如果你想每天学习一个知识点，关注我的【公】【众】【号】【小白debug】。




