___
## 概述

Context是Go中用于管理并发goroutine的机制, 能够优雅的实现消息通知, 级联中止等特性

## 接口定义

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key any) any
}
```
ctx的接口定义非常简单, 有如下四个方法
- Deadline
	返回绑定当前context的任务被取消的时间, 若没有则ok为false
- Done
	当绑定当前context的任务被取消时返回一个关闭的channel, 如果当前任务不会被关闭则返回一个nil
- Err
	如果Done的channel没有关闭则返回nil, 若关闭了则返回对应的原因: Canceled或DeadlineExceeded, 返回过非空值后再次调用会返回相同的error
- Value
	根据key返回context中存储的value, 不存在则返回nil

## emptyCtx

```go
type emptyCtx struct{}

func (emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (emptyCtx) Done() <-chan struct{} {
	return nil
}

func (emptyCtx) Err() error {
	return nil
}

func (emptyCtx) Value(key any) any {
	return nil
}
```
emptyCtx是一个struct{}类型的变量, 实现了Context接口, 不会被取消, 没有value, 没有deadline.
同时作为backgroundCtx和todoCtx的公共基础

## Background

```go
type backgroundCtx struct{ emptyCtx }

func (backgroundCtx) String() string {
	return "context.Background"
}

func Background() Context {  
    return backgroundCtx{}  
}
```
`Background()`方法返回一个空的Context实例, 这个实例没有值, 不会被取消, 也没有deadline. 常用于main函数中, 作为顶级的Context或初始化

## TODO

```go
type todoCtx struct{ emptyCtx }  
  
func (todoCtx) String() string {  
    return "context.TODO"  
}

func TODO() Context {  
    return todoCtx{}  
}
```
`TODO()`方法返回一个空的Context实例,  这个实例同样没有值, 不会被取消, 也没有deadlines. 当不知道使用哪个Context或者暂时还没有使用Context机制时使用

## Value

```go
type valueCtx struct {  
    Context  
    key, val any  
}

func stringify(v any) string {  
    switch s := v.(type) {  
    case stringer:  
       return s.String()  
    case string:  
       return s  
    }  
    return "<not Stringer>"  
}

func (c *valueCtx) String() string {  
    return contextName(c.Context) + ".WithValue(type " +  
       reflectlite.TypeOf(c.key).String() +  
       ", val " + stringify(c.val) + ")"  
}

func (c *valueCtx) Value(key any) any {  
    if c.key == key {  
       return c.val  
    }  
    return value(c.Context, key)  
}

func value(c Context, key any) any {
	for {
		switch ctx := c.(type) {
		case *valueCtx:
			if key == ctx.key {
				return ctx.val
			}
			c = ctx.Context
		case *cancelCtx:
			if key == &cancelCtxKey {
				return c
			}
			c = ctx.Context
		case withoutCancelCtx:
			if key == &cancelCtxKey {
				// This implements Cause(ctx) == nil
				// when ctx is created using WithoutCancel.
				return nil
			}
			c = ctx.c
		case *timerCtx:
			if key == &cancelCtxKey {
				return &ctx.cancelCtx
			}
			c = ctx.Context
		case backgroundCtx, todoCtx:
			return nil
		default:
			return c.Value(key)
		}
	}
}

func WithValue(parent Context, key, val any) Context {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
```
- valueCtx使用一个Context类型的变量来表示父节点, 所以当前节点继承了父节点的所有信息, 同时valueCtx还携带了一组键值对, 可以携带额外的信息
- valueCtx实现了Value方法可以在Context链路上查找需要的key
- WithValue可以向ctx中添加键值对, 这里的添加不是在原context上直接添加, 而是将键值对添加在子节点上, 形成一个Context链路
## Cancel

```go
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     atomic.Value          // of chan struct{}, created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
	cause    error                 // set to non-nil by the first cancel call
}

type canceler interface {  
    cancel(removeFromParent bool, err, cause error)  
    Done() <-chan struct{}  
}

func (c *cancelCtx) Value(key any) any {
	if key == &cancelCtxKey {
		return c
	}
	return value(c.Context, key)
}

func (c *cancelCtx) Done() <-chan struct{} {
	d := c.done.Load()
	if d != nil {
		return d.(chan struct{})
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	d = c.done.Load()
	if d == nil {
		d = make(chan struct{})
		c.done.Store(d)
	}
	return d.(chan struct{})
}

func (c *cancelCtx) Err() error {
	c.mu.Lock()
	err := c.err
	c.mu.Unlock()
	return err
}

func (c *cancelCtx) String() string {
	return contextName(c.Context) + ".WithCancel"
}

func (c *cancelCtx) cancel(removeFromParent bool, err, cause error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	if cause == nil {
		cause = err
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
	c.cause = cause
	d, _ := c.done.Load().(chan struct{})
	if d == nil {
		c.done.Store(closedchan)
	} else {
		close(d)
	}
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err, cause)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```
cancelCtx有一个Context父节点, done表示一个channel用于传递关闭信号, children维护子节点, err则存储任务结束的原因
当执行cancel时, 会将done设置为一个关闭的channel, 并将子节点依次取消, 如果有需要还会将当前节点从父节点上移除
## WithCancel
```go
var closedchan = make(chan struct{})

func init() {  
    close(closedchan)  
}

type CancelFunc func()

func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := withCancel(parent)
	return c, func() { c.cancel(true, Canceled, nil) }
}

type CancelCauseFunc func(cause error)

func WithCancelCause(parent Context) (ctx Context, cancel CancelCauseFunc) {  
    c := withCancel(parent)  
    return c, func(cause error) { c.cancel(true, Canceled, cause) }  
}

func withCancel(parent Context) *cancelCtx {  
    if parent == nil {  
       panic("cannot create context from nil parent")  
    }  
    c := &cancelCtx{}  
    c.propagateCancel(parent, c)  
    return c  
}

func (c *cancelCtx) propagateCancel(parent Context, child canceler) {  
    c.Context = parent  
  
    done := parent.Done()  
    if done == nil {  
       return // parent is never canceled  
    }  
  
    select {  
    case <-done:  
       // parent is already canceled  
       child.cancel(false, parent.Err(), Cause(parent))  
       return  
    default:  
    }  
  
    if p, ok := parentCancelCtx(parent); ok {  
       // parent is a *cancelCtx, or derives from one.  
       p.mu.Lock()  
       if p.err != nil {  
          // parent has already been canceled  
          child.cancel(false, p.err, p.cause)  
       } else {  
          if p.children == nil {  
             p.children = make(map[canceler]struct{})  
          }  
          p.children[child] = struct{}{}  
       }  
       p.mu.Unlock()  
       return  
    }  
  
    if a, ok := parent.(afterFuncer); ok {  
       // parent implements an AfterFunc method.  
       c.mu.Lock()  
       stop := a.AfterFunc(func() {  
          child.cancel(false, parent.Err(), Cause(parent))  
       })  
       c.Context = stopCtx{  
          Context: parent,  
          stop:    stop,  
       }  
       c.mu.Unlock()  
       return  
    }  
  
    goroutines.Add(1)  
    go func() {  
       select {  
       case <-parent.Done():  
          child.cancel(false, parent.Err(), Cause(parent))  
       case <-child.Done():  
       }  
    }()  
}

func Cause(c Context) error {
	if cc, ok := c.Value(&cancelCtxKey).(*cancelCtx); ok {
		cc.mu.Lock()
		defer cc.mu.Unlock()
		return cc.cause
	}
	return c.Err()
}
```
WithCancel创建一个可以取消的context, 即cancelCtx类型的context, 返回一个context和一个CancelFunc, 调用CancelFunc就可以触发Cancel操作
## timerCtx

```go
type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}

func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
	return c.deadline, true
}

func (c *timerCtx) String() string {
	return contextName(c.cancelCtx.Context) + ".WithDeadline(" +
		c.deadline.String() + " [" +
		time.Until(c.deadline).String() + "])"
}

func (c *timerCtx) cancel(removeFromParent bool, err, cause error) {
	c.cancelCtx.cancel(false, err, cause)
	if removeFromParent {
		// Remove this timerCtx from its parent cancelCtx's children.
		removeChild(c.cancelCtx.Context, c)
	}
	c.mu.Lock()
	if c.timer != nil {
		c.timer.Stop()
		c.timer = nil
	}
	c.mu.Unlock()
}
```

timerCtx是可以定时取消的Ctx, 当定时器触发时将内部的cancelCtx取消, 然后从cancelCtx的祖先节点上移除, 最后取消计时器
## WithTimeout & WithDeadline

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}

func WithTimeoutCause(parent Context, timeout time.Duration, cause error) (Context, CancelFunc) {
	return WithDeadlineCause(parent, time.Now().Add(timeout), cause)
}

func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	return WithDeadlineCause(parent, d, nil)
}

func WithDeadlineCause(parent Context, d time.Time, cause error) (Context, CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// The current deadline is already sooner than the new one.
		return WithCancel(parent)
	}
	c := &timerCtx{
		deadline: d,
	}
	c.cancelCtx.propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded, cause) // deadline has already passed
		return c, func() { c.cancel(false, Canceled, nil) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded, cause)
		})
	}
	return c, func() { c.cancel(true, Canceled, nil) }
}
```
Deadline是接受时间点
Timeout是接受时间长度
## After

```go
func AfterFunc(ctx Context, f func()) (stop func() bool) {
	a := &afterFuncCtx{
		f: f,
	}
	a.cancelCtx.propagateCancel(ctx, a)
	return func() bool {
		stopped := false
		a.once.Do(func() {
			stopped = true
		})
		if stopped {
			a.cancel(true, Canceled, nil)
		}
		return stopped
	}
}

type afterFuncer interface {  
    AfterFunc(func()) func() bool  
}

type afterFuncCtx struct {  
    cancelCtx  
    once sync.Once // either starts running f or stops f from running    f    func()  
}

func (a *afterFuncCtx) cancel(removeFromParent bool, err, cause error) {  
    a.cancelCtx.cancel(false, err, cause)  
    if removeFromParent {  
       removeChild(a.Context, a)  
    }  
    a.once.Do(func() {  
       go a.f()  
    })  
}
```
在context取消后再执行afterFunc