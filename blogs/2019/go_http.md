---
title: Go http 解读
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - golang
tags:
 - golang
 - http
publish: true
---

从最简单的http server 开始读

```go
func main() {
	http.HandleFunc("/home", func(writer http.ResponseWriter, request *http.Request) {
		writer.Write([]byte("hello"))
	})

	log.Fatal(http.ListenAndServe(":8888", nil))
}
```

先看 `http.HandleFunc` 

```go
// 没有自己的 server mux 系统会去使用默认的, 所以这里你其实可以自定义一个 ServerMux
var DefaultServeMux = &defaultServeMux

var defaultServeMux ServeMux

func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}

type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry
	es    []muxEntry // slice of entries sorted from longest to shortest.
	hosts bool       // whether any patterns contain hostnames
}

type muxEntry struct {
	h       Handler
	pattern string
}
```

DefaultServeMux.HandleFunc 往下就是注册 handler 了

```go
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
  mux.Handle(pattern, HandlerFunc(handler)) // 这里强制类型转换，让其(方法)拥有了 ServeHTTP 方法
}

type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

继续往下

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}

// 可以看到第二个参数需要实现 Handler 接口，由此可见上面的强制转换是为了适配，源码注释也讲到了这是一个适配器
func (mux *ServeMux) Handle(pattern string, handler Handler) {
  // 线程安全
	mux.mu.Lock()
	defer mux.mu.Unlock()

	if pattern == "" {
		panic("http: invalid pattern")
	}
	if handler == nil {
		panic("http: nil handler")
	}
  // 不能重复注册
	if _, exist := mux.m[pattern]; exist {
		panic("http: multiple registrations for " + pattern)
	}

  // 判断；初始化 ServeMux 的 m 属性
	if mux.m == nil {
		mux.m = make(map[string]muxEntry)
	}
  // 生成 Entry
	e := muxEntry{h: handler, pattern: pattern}
  // 注册
	mux.m[pattern] = e
  // 如果 pattern 以 "/" 结尾，比如 /home/ , /abc/，则把 entry 加入到 mux 的es属性中，这个用来匹配目前前缀的路径，比如 e.pattern /home/ => 以 “/” 结尾，那么就会被加入到 mux.es 中, 之后在访问 /home/asgasdh 都会被命中这个路由
	if pattern[len(pattern)-1] == '/' {
		mux.es = appendSorted(mux.es, e)
	}

  // 很简单的判断了下，如果pattern第一个不是/那么hosts设为true，目前还不知道hosts什么用⁉️
	if pattern[0] != '/' {
		mux.hosts = true
	}
}


// sort.Search 找出符合条件的最小值
func appendSorted(es []muxEntry, e muxEntry) []muxEntry {
	n := len(es)
	i := sort.Search(n, func(i int) bool {
		return len(es[i].pattern) < len(e.pattern)
	})
  
  // i == n 场景是，例如 e.pattern 的长度比 es 中的任意一个pattern都要短，那它就是最后一个
  // es 里面的结构大概是这样的 [{h:"", pattern:"/home"} {h:"", pattern:"/abc"}, {h:"", pattern:"/a"}] 按长度倒序排列的
	if i == n {
		return append(es, e)
	}
	
	// 把 e 插入相应的位置
	es = append(es, muxEntry{})
  // copy 会根据es的长度进行复制，溢出的直接抛弃，所以上面的 append 要预留出一个位置给 copy
	copy(es[i+1:], es[i:])
  // 把 e 插入相应的位置
	es[i] = e
	return es
}
```

然后再看 `log.Fatal(http.ListenAndServe(":8888", nil))`

```go
// 看到这里只传入了 地址和 handler(nil)
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}

// 完整的 Server结构体
type Server struct {
	Addr    string
	Handler Handler

	TLSConfig *tls.Config

	ReadTimeout time.Duration

	ReadHeaderTimeout time.Duration

	WriteTimeout time.Duration

	IdleTimeout time.Duration

	MaxHeaderBytes int

	TLSNextProto map[string]func(*Server, *tls.Conn, Handler)
  
	ConnState func(net.Conn, ConnState)

	ErrorLog *log.Logger

	BaseContext func(net.Listener) context.Context

	ConnContext func(ctx context.Context, c net.Conn) context.Context

	disableKeepAlives int32     // accessed atomically.
	inShutdown        int32     // 非零意味着服务以及关闭
	nextProtoOnce     sync.Once // guards setupHTTP2_* init
	nextProtoErr      error     // result of http2.ConfigureServer if used

	mu         sync.Mutex
	listeners  map[*net.Listener]struct{}
	activeConn map[*conn]struct{}
	doneChan   chan struct{}
	onShutdown []func()
}


------ 
这里看出你其实可以自定义 Server
srv := http.Server{
		Addr:              ":8888",
		Handler:           nil,
		TLSConfig:         nil,
		ReadTimeout:       0,
		ReadHeaderTimeout: 0,
		WriteTimeout:      0,
		IdleTimeout:       0,
		MaxHeaderBytes:    0,
		TLSNextProto:      nil,
		ConnState:         nil,
		ErrorLog:          nil,
		BaseContext:       nil,
		ConnContext:       nil,
	}
	log.Fatal(srv.ListenAndServe())
```

继续

```go
func (srv *Server) ListenAndServe() error {
  // 判断服务器是否已经 关闭，如果关闭则返回 ErrServerClosed，这里应该是用来平滑关闭服务的(graceful shutdown)
	if srv.shuttingDown() {
		return ErrServerClosed
	}
	addr := srv.Addr
  // 这里如果addr不传默认为80
  // 太底层了，看不懂了，勉强看清楚返回了80😅
	if addr == "" {
		addr = ":http"
	}
  // 调用 net 包的 listen
	ln, err := net.Listen("tcp", addr)
	if err != nil {
		return err
	}
	return srv.Serve(ln)
}
```

继续

```go
type Listener interface {
	Accept() (Conn, error)

	Close() error

	Addr() Addr
}

func (srv *Server) Serve(l net.Listener) error {
  // 看起来好像是专门给测试用的，不太清楚这个
	if fn := testHookServerServe; fn != nil {
		fn(srv, l)
	}

  // 把原 Listener 结构体赋值给一个副本
	origListener := l
  // 只调用一次关闭
	l = &onceCloseListener{Listener: l}
	defer l.Close()

  // 如果开启了http2就用http2，暂时不分析
	if err := srv.setupHTTP2_Serve(); err != nil {
		return err
	}

  // 设置track日志，跟踪服务轨迹
	if !srv.trackListener(&l, true) {
		return ErrServerClosed
	}
  // server退出后清除，让gc回收
	defer srv.trackListener(&l, false)

	var tempDelay time.Duration // how long to sleep on accept failure

  // base 上下文
	baseCtx := context.Background()
  // 如果 服务的BaseContext不为空那么再设置BaseContext，带上上一个listener
	if srv.BaseContext != nil {
		baseCtx = srv.BaseContext(origListener)
		if baseCtx == nil {
			panic("BaseContext returned a nil context")
		}
	}

  // 封装一个带 k/v 的 ctx
  // key 是 contextKey 结构体指针
  // type contextKey struct {
  //   name string
  // }
  // value 是 Server 指针
  
	ctx := context.WithValue(baseCtx, ServerContextKey, srv)
	for {
    // 接收连接
		rw, e := l.Accept()
		if e != nil {
			select {
        // Server 里面还有个属性是 doneChan 
        // 和 inShutdown 作用大致相同，也是判断服务器是否关闭，关闭就返回 ErrServerClosed ，为啥要有这个呢？再往下看看
			case <-srv.getDoneChan():
				return ErrServerClosed
			default:
			}
      // 如果出错并且服务器没关闭，继续往下走

      // 判断 e 是不是 实现了 net.Error 接口，并且e是不是临时的错误，比如超时 timeoutError
			if ne, ok := e.(net.Error); ok && ne.Temporary() {
        //  tempDelay 上文定义了该变量，默认是 0，
        // 初次报错，设置延迟是 5 毫秒，后来每次 * 2
				if tempDelay == 0 {
					tempDelay = 5 * time.Millisecond
				} else {
					tempDelay *= 2
				}
        // 最大超时时间到1秒后，维持1秒
				if max := 1 * time.Second; tempDelay > max {
					tempDelay = max
				}
        // 输出错误 睡 tempDelay 秒
        // 高并发下应该能出现这个错误？涉及到 net 包，目前看不懂暂时跳过😂
				srv.logf("http: Accept error: %v; retrying in %v", e, tempDelay)
				time.Sleep(tempDelay)
				continue
			}
			return e
		}
		connCtx := ctx
    // srv.ConnContext 回调提供了修改 ctx 的能力
		if cc := srv.ConnContext; cc != nil {
			connCtx = cc(connCtx, rw)
			if connCtx == nil {
				panic("ConnContext returned nil")
			}
		}
    // 重置延迟时间
		tempDelay = 0
		c := srv.newConn(rw) // 获取一个新的 conn 指针
		c.setState(c.rwc, StateNew) // conn 链路追踪, 触发状态回调钩子
    // 开启协程处理请求
		go c.serve(connCtx)
	}
}

func (srv *Server) newConn(rwc net.Conn) *conn {
  // 封装了一个结构体
	c := &conn{
		server: srv,
		rwc:    rwc,
	}
  // 是否记录连接
  // 返回一个带名字的 net.Conn
	if debugServerConnections {
		c.rwc = newLoggingConn("server", c.rwc)
	}
	return c
}

func (s *Server) trackConn(c *conn, add bool) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if s.activeConn == nil {
		s.activeConn = make(map[*conn]struct{})
	}
	if add {
    // server 记录 conn连接
		s.activeConn[c] = struct{}{}
	} else {
    // server 删除 conn连接
		delete(s.activeConn, c)
	}
}

func (c *conn) setState(nc net.Conn, state ConnState) {
	srv := c.server
	switch state {
	case StateNew:
		srv.trackConn(c, true)
	case StateHijacked, StateClosed:
		srv.trackConn(c, false)
	}
	if state > 0xff || state < 0 {
		panic("internal error")
	}
	packedState := uint64(time.Now().Unix()<<8) | uint64(state)
  // 记录当前时间戳*2的8次方？why 为啥乘以2的8次？
	atomic.StoreUint64(&c.curState.atomic, packedState)
  // 连接状态发生改变时触发，调用 ConnState 钩子回调
	if hook := srv.ConnState; hook != nil {
		hook(nc, state)
	}
}

type conn struct {
	// server is the server on which the connection arrived.
	// Immutable; never nil.
	server *Server

	// cancelCtx cancels the connection-level context.
	cancelCtx context.CancelFunc

	// rwc is the underlying network connection.
	// This is never wrapped by other types and is the value given out
	// to CloseNotifier callers. It is usually of type *net.TCPConn or
	// *tls.Conn.
	rwc net.Conn

	// remoteAddr is rwc.RemoteAddr().String(). It is not populated synchronously
	// inside the Listener's Accept goroutine, as some implementations block.
	// It is populated immediately inside the (*conn).serve goroutine.
	// This is the value of a Handler's (*Request).RemoteAddr.
	remoteAddr string

	// tlsState is the TLS connection state when using TLS.
	// nil means not TLS.
	tlsState *tls.ConnectionState

	// werr is set to the first write error to rwc.
	// It is set via checkConnErrorWriter{w}, where bufw writes.
	werr error

	// r is bufr's read source. It's a wrapper around rwc that provides
	// io.LimitedReader-style limiting (while reading request headers)
	// and functionality to support CloseNotifier. See *connReader docs.
	r *connReader

	// bufr reads from r.
	bufr *bufio.Reader

	// bufw writes to checkConnErrorWriter{c}, which populates werr on error.
	bufw *bufio.Writer

	// lastMethod is the method of the most recent request
	// on this connection, if any.
	lastMethod string

	curReq atomic.Value // of *response (which has a Request in it)

	curState struct{ atomic uint64 } // packed (unixtime<<8|uint8(ConnState))

	// mu guards hijackedv
	mu sync.Mutex

	// hijackedv is whether this connection has been hijacked
	// by a Handler with the Hijacker interface.
	// It is guarded by mu.
	hijackedv bool
}


const (
	// 代表新连接
	StateNew ConnState = iota

	// StateActive represents a connection that has read 1 or more
	// bytes of a request. The Server.ConnState hook for
	// StateActive fires before the request has entered a handler
	// and doesn't fire again until the request has been
	// handled. After the request is handled, the state
	// transitions to StateClosed, StateHijacked, or StateIdle.
	// For HTTP/2, StateActive fires on the transition from zero
	// to one active request, and only transitions away once all
	// active requests are complete. That means that ConnState
	// cannot be used to do per-request work; ConnState only notes
	// the overall state of the connection.
	StateActive

	// StateIdle represents a connection that has finished
	// handling a request and is in the keep-alive state, waiting
	// for a new request. Connections transition from StateIdle
	// to either StateActive or StateClosed.
	StateIdle

	// StateHijacked represents a hijacked connection.
	// This is a terminal state. It does not transition to StateClosed.
	StateHijacked

	// StateClosed represents a closed connection.
	// This is a terminal state. Hijacked connections do not
	// transition to StateClosed.
	StateClosed
)

```

处理每个连接

```go
func (c *conn) serve(ctx context.Context) {
  // 获取访问地址
	c.remoteAddr = c.rwc.RemoteAddr().String()
  // 又包装了一个 k/v k=>&contextKey{"local-addr"} v => addr
	ctx = context.WithValue(ctx, LocalAddrContextKey, c.rwc.LocalAddr())
  
  // 捕获每个请求的异常, 防止某个协程报错导致server退出
	defer func() {
		if err := recover(); err != nil && err != ErrAbortHandler {
			const size = 64 << 10
			buf := make([]byte, size)
      // runtime.Stack 有待研究⁉️
			buf = buf[:runtime.Stack(buf, false)]
      // 输出错误内容
			c.server.logf("http: panic serving %v: %v\n%s", c.remoteAddr, err, buf)
		}
    // 连接是否被劫持
    // 是：关闭连接，设置状态触发回调
    // 是吗情况会被劫持，有待研究⁉️
		if !c.hijacked() {
			c.close()
			c.setState(c.rwc, StateClosed)
		}
	}()

	if tlsConn, ok := c.rwc.(*tls.Conn); ok {
		if d := c.server.ReadTimeout; d != 0 {
      // 设置 net.Conn 读取数据的超时时间
			c.rwc.SetReadDeadline(time.Now().Add(d))
		}
		if d := c.server.WriteTimeout; d != 0 {
      // 设置 net.Conn 写入数据的超时时间
			c.rwc.SetWriteDeadline(time.Now().Add(d))
		}
    
    // https 握手失败直接返回
    // 这一段看起来很吃力，先跳过
		if err := tlsConn.Handshake(); err != nil {
			if re, ok := err.(tls.RecordHeaderError); ok && re.Conn != nil && tlsRecordHeaderLooksLikeHTTP(re.RecordHeader) {
				io.WriteString(re.Conn, "HTTP/1.0 400 Bad Request\r\n\r\nClient sent an HTTP request to an HTTPS server.\n")
				re.Conn.Close()
				return
			}
			c.server.logf("http: TLS handshake error from %s: %v", c.rwc.RemoteAddr(), err)
			return
		}
		c.tlsState = new(tls.ConnectionState)
		*c.tlsState = tlsConn.ConnectionState()
		if proto := c.tlsState.NegotiatedProtocol; validNPN(proto) {
			if fn := c.server.TLSNextProto[proto]; fn != nil {
				h := initNPNRequest{ctx, tlsConn, serverHandler{c.server}}
				fn(c.server, tlsConn, h)
			}
			return
		}
	}

	// HTTP/1.x from here on.

  // 包装一个可取消的ctx
	ctx, cancelCtx := context.WithCancel(ctx)
	c.cancelCtx = cancelCtx
	defer cancelCtx()

  // 设置连接属性
	c.r = &connReader{conn: c}
  // 默认 4096 size 的 bf reader
	c.bufr = newBufioReader(c.r)
  // 4k bf writer, checkConnErrorWriter 适配 io.Writer
	c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)

  
  // 开始处理请求
	for {
    // 这里内容很多，下面详细分解
		w, err := c.readRequest(ctx)
		...
	}
}

//堆栈将调用goroutine的堆栈跟踪格式格式化为buf
//并返回写入buf的字节数。
//如果全部为真，则堆栈格式将堆栈所有其他goroutine的跟踪
//在当前goroutine的跟踪之后放入buf。
func Stack(buf []byte, all bool) int {
	if all {
		stopTheWorld("stack trace")
	}

	n := 0
	if len(buf) > 0 {
		gp := getg()
		sp := getcallersp()
		pc := getcallerpc()
		systemstack(func() {
			g0 := getg()
			// Force traceback=1 to override GOTRACEBACK setting,
			// so that Stack's results are consistent.
			// GOTRACEBACK is only about crash dumps.
			g0.m.traceback = 1
			g0.writebuf = buf[0:0:len(buf)]
			goroutineheader(gp)
			traceback(pc, sp, 0, gp)
			if all {
				tracebackothers(gp)
			}
			g0.m.traceback = 0
			n = len(g0.writebuf)
			g0.writebuf = nil
		})
	}

	if all {
		startTheWorld()
	}
	return n
}
```

处理请求详细过程

```go
func (c *conn) readRequest(ctx context.Context) (w *response, err error) {
  // 判断是否被劫持？什么情况会发生，还不知道🧐
	if c.hijacked() {
		return nil, ErrHijacked
	}

	var (
		wholeReqDeadline time.Time // or zero if none
		hdrDeadline      time.Time // or zero if none
	)
  
  // 请求开始的时间
	t0 := time.Now()
  // 设置两个超时时间
	if d := c.server.readHeaderTimeout(); d != 0 {
		hdrDeadline = t0.Add(d)
	}
	if d := c.server.ReadTimeout; d != 0 {
		wholeReqDeadline = t0.Add(d)
	}
  // 设置读取时间
	c.rwc.SetReadDeadline(hdrDeadline)
  // 这里为啥和上面设置超时时间不一样，defer目的是啥？
	if d := c.server.WriteTimeout; d != 0 {
		defer func() {
			c.rwc.SetWriteDeadline(time.Now().Add(d))
		}()
	}

  // 设置 1M+4096的读取字节限制
	c.r.setReadLimit(c.server.initialReadLimitSize())
	if c.lastMethod == "POST" {
		// RFC 7230 section 3 tolerance for old buggy clients.
		peek, _ := c.bufr.Peek(4) // ReadRequest will get err below
		c.bufr.Discard(numLeadingCRorLF(peek))
	}
	req, err := readRequest(c.bufr, keepHostHeader)
	if err != nil {
		if c.r.hitReadLimit() {
			return nil, errTooLarge
		}
		return nil, err
	}

	if !http1ServerSupportsRequest(req) {
		return nil, badRequestError("unsupported protocol version")
	}

	c.lastMethod = req.Method
	c.r.setInfiniteReadLimit()

	hosts, haveHost := req.Header["Host"]
	isH2Upgrade := req.isH2Upgrade()
	if req.ProtoAtLeast(1, 1) && (!haveHost || len(hosts) == 0) && !isH2Upgrade && req.Method != "CONNECT" {
		return nil, badRequestError("missing required Host header")
	}
	if len(hosts) > 1 {
		return nil, badRequestError("too many Host headers")
	}
	if len(hosts) == 1 && !httpguts.ValidHostHeader(hosts[0]) {
		return nil, badRequestError("malformed Host header")
	}
	for k, vv := range req.Header {
		if !httpguts.ValidHeaderFieldName(k) {
			return nil, badRequestError("invalid header name")
		}
		for _, v := range vv {
			if !httpguts.ValidHeaderFieldValue(v) {
				return nil, badRequestError("invalid header value")
			}
		}
	}
	delete(req.Header, "Host")

	ctx, cancelCtx := context.WithCancel(ctx)
	req.ctx = ctx
	req.RemoteAddr = c.remoteAddr
	req.TLS = c.tlsState
	if body, ok := req.Body.(*body); ok {
		body.doEarlyClose = true
	}

	// Adjust the read deadline if necessary.
	if !hdrDeadline.Equal(wholeReqDeadline) {
		c.rwc.SetReadDeadline(wholeReqDeadline)
	}

	w = &response{
		conn:          c,
		cancelCtx:     cancelCtx,
		req:           req,
		reqBody:       req.Body,
		handlerHeader: make(Header),
		contentLength: -1,
		closeNotifyCh: make(chan bool, 1),

		// We populate these ahead of time so we're not
		// reading from req.Header after their Handler starts
		// and maybe mutates it (Issue 14940)
		wants10KeepAlive: req.wantsHttp10KeepAlive(),
		wantsClose:       req.wantsClose(),
	}
	if isH2Upgrade {
		w.closeAfterReply = true
	}
	w.cw.res = w
	w.w = newBufioWriterSize(&w.cw, bufferBeforeChunkingSize)
	return w, nil
}
```



前面好大一段都读不懂，大概就是处理http协议，判断header的合法性，封装request和response吧（第一遍读源码不要求啥都看的懂😂），不过终于来到这里

```go
serverHandler{c.server}.ServeHTTP(w, w.req)
 
// 一开始 	log.Fatal(http.ListenAndServe(":8888", nil)) handler 是 nil，
// 所以使用 DefaultServeMux

func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
	handler := sh.srv.Handler
	if handler == nil {
		handler = DefaultServeMux
	}
	if req.RequestURI == "*" && req.Method == "OPTIONS" {
		handler = globalOptionsHandler{}
	}
	handler.ServeHTTP(rw, req)
}
```

```go
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
	if r.RequestURI == "*" {
		if r.ProtoAtLeast(1, 1) {
			w.Header().Set("Connection", "close")
		}
		w.WriteHeader(StatusBadRequest)
		return
	}
	h, _ := mux.Handler(r)
	h.ServeHTTP(w, r)
}

func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {

	if r.Method == "CONNECT" {
		if u, ok := mux.redirectToPathSlash(r.URL.Host, r.URL.Path, r.URL); ok {
			return RedirectHandler(u.String(), StatusMovedPermanently), u.Path
		}

		return mux.handler(r.Host, r.URL.Path)
	}

	host := stripHostPort(r.Host)
	path := cleanPath(r.URL.Path) // 使用了 path.clean 详见下文

  // 这里判断是否需要重定向，比如 注册的时候是 /home/ ，而 request path 是 /home ,那么这种情况下需要重定向到  /home/
	if u, ok := mux.redirectToPathSlash(host, path, r.URL); ok {
		return RedirectHandler(u.String(), StatusMovedPermanently), u.Path
	}

  // 如果处理之后的路径和原路径不一样，会返回一个重定向 handler
	if path != r.URL.Path {
		_, pattern = mux.handler(host, path)
		url := *r.URL
		url.Path = path
		return RedirectHandler(url.String(), StatusMovedPermanently), pattern
	}

	return mux.handler(host, r.URL.Path)
}
```

```go
func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
    mux.mu.RLock()
   defer mux.mu.RUnlock()

   // Host-specific pattern takes precedence over generic ones
   if mux.hosts {
      h, pattern = mux.match(host + path)
   }
   if h == nil {
      h, pattern = mux.match(path)
   }
   if h == nil {
     // 啥handler都没找到的话直接返回 NotFoundHandler
      h, pattern = NotFoundHandler(), ""
   }
   return
}

func NotFound(w ResponseWriter, r *Request) { Error(w, "404 page not found", StatusNotFound) }


// 到这里，mux就返回了注册时的 handler ，也就是之后的逻辑就是 你自己写的handler的逻辑了
func (mux *ServeMux) match(path string) (h Handler, pattern string) {
  // 如果路径精确匹配 handler 则直接返回handler
	v, ok := mux.m[path]
	if ok {
		return v.h, v.pattern
	}

  // 如果没找到，则找前缀 /home/1234 会命中 /home/ 路径 
	for _, e := range mux.es {
		if strings.HasPrefix(path, e.pattern) {
			return e.h, e.pattern
		}
	}
	return nil, ""
}

```

在之后就是完成request了



## 课外 path 包

```go

func main() {
	paths := []string{
		"a/c",
		"a//c",
		"a/c/.",
		"a/c/b/..",
		"/../a/c",
		"/../a/b/../././/c",
	}

	for _, p := range paths {
		fmt.Printf("Clean(%q) = %q\n", p, path.Clean(p))
	}

}

lean("a/c") = "a/c"
Clean("a//c") = "a/c"
Clean("a/c/.") = "a/c"
Clean("a/c/b/..") = "a/c"
Clean("/../a/c") = "/a/c"
Clean("/../a/b/../././/c") = "/a/c"
```


