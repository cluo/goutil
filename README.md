# goutil   [![GoDoc](https://godoc.org/github.com/tsuna/gohbase?status.png)](https://godoc.org/github.com/henrylee2cn/goutil)

Common and useful utils for the Go project development.

## 1. Inclusion criteria

- Only rely on the Go standard package
- Functions or lightweight packages
- Non-business related general tools

## 2. Contents

- [Calendar](#calendar) Chinese Lunar Calendar, Solar Calendar and cron time rules
- [CoarseTime](#coarsetime) Current time truncated to the nearest second
- [GoPool](#gopool) Goroutines' pool
- [ResPool](#respool) Resources' pool
- [Shutdown](#shutdown) Close current program gracefully
- [Various](#various) Various small functions


## 3. UtilsAPI

### Calendar

Chinese Lunar Calendar, Solar Calendar and cron time rules.

- import it

	```go
	"github.com/henrylee2cn/goutil/calendar"
	```

[Calendar details](calendar/README.md)

### CoarseTime

The current time truncated to the nearest second.

- import it

	```go
	"github.com/henrylee2cn/goutil/coarsetime"
	```

- CoarseTimeNow returns the current time truncated to the nearest second.
This is a faster alternative to time.Now().

	```go
	func CoarseTimeNow() time.Time
	```

### GoPool

GoPool is a Goroutines pool. It can control concurrent numbers, reuse goroutines.

- import it

	```go
	"github.com/henrylee2cn/goutil/pool"
	```

- GoPool executes concurrently incoming function via a pool of goroutines in
FILO order, i.e. the most recently stopped goroutine will execute the next
incoming function.
Such a scheme keeps CPU caches hot (in theory).

	```go
	type GoPool struct {
		// Has unexported fields.
	}
	```
	    
- NewGoPool creates a new *GoPool.
If maxGoroutinesAmount<=0, will use default value.
If maxGoroutineIdleDuration<=0, will use default value.

	```go
	func NewGoPool(maxGoroutinesAmount int, maxGoroutineIdleDuration time.Duration) *GoPool
	```

- Go executes function via a goroutine.
If returns non-nil, the function cannot be executed because exceeded maxGoroutinesAmount limit.

	```go
	func (gp *GoPool) Go(fn func()) error
	```

- Stop starts GoPool.
If calling 'Go' after calling 'Stop', will no longer reuse goroutine.

	```go
	func (gp *GoPool) Stop()
	```

### ResPool

ResPool is a high availability/high concurrent resource pool, which automatically manages the number of resources.
So it is similar to database/sql's db pool.

- import it

	```go
	"github.com/henrylee2cn/goutil/pool"
	```

- ResPool is a pool of zero or more underlying avatar(resource).
It's safe for concurrent use by multiple goroutines.
ResPool creates and frees resource automatically;
it also maintains a free pool of idle avatar(resource).

	```go
	type ResPool interface {
		// Name returns the name.
		Name() string
		// Get returns a object in Resource type.
		Get() (Resource, error)
		// GetContext returns a object in Resource type.
		// Support context cancellation.
		GetContext(context.Context) (Resource, error)
		// Put gives a resource back to the ResPool.
		// If error is not nil, close the avatar.
		Put(Resource, error)
		// Callback callbacks your handle function, returns the error of getting resource or handling.
		// Support recover panic.
		Callback(func(Resource) error) error
		// Callback callbacks your handle function, returns the error of getting resource or handling.
		// Support recover panic and context cancellation.
		CallbackContext(context.Context, func(Resource) error) error
		// SetMaxLifetime sets the maximum amount of time a resource may be reused.
		//
		// Expired resource may be closed lazily before reuse.
		//
		// If d <= 0, resource are reused forever.
		SetMaxLifetime(d time.Duration)
		// SetMaxIdle sets the maximum number of resources in the idle
		// resource pool.
		//
		// If SetMaxIdle is greater than 0 but less than the new MaxIdle
		// then the new MaxIdle will be reduced to match the SetMaxIdle limit
		//
		// If n <= 0, no idle resources are retained.
		SetMaxIdle(n int)
		// SetMaxOpen sets the maximum number of open resources.
		//
		// If MaxIdle is greater than 0 and the new MaxOpen is less than
		// MaxIdle, then MaxIdle will be reduced to match the new
		// MaxOpen limit
		//
		// If n <= 0, then there is no limit on the number of open resources.
		// The default is 0 (unlimited).
		SetMaxOpen(n int)
		// Close closes the ResPool, releasing any open resources.
		//
		// It is rare to close a ResPool, as the ResPool handle is meant to be
		// long-lived and shared between many goroutines.
		Close() error
		// Stats returns resource statistics.
		Stats() ResPoolStats
	}
	```

- NewResPool creates ResPool.

	```go
	func NewResPool(name string, newfunc func(context.Context) (Resource, error)) ResPool
	```

- Resource is a resource that can be stored in the ResPool.

	```go
	type Resource interface {
		// SetAvatar stores the contact with resPool
		// Do not call it yourself, it is only called by (*ResPool).get, and will only be called once
		SetAvatar(*Avatar)
		// GetAvatar gets the contact with resPool
		// Do not call it yourself, it is only called by (*ResPool).Put
		GetAvatar() *Avatar
		// Close closes the original source
		// No need to call it yourself, it is only called by (*Avatar).close
		Close() error
	}
	```

- Avatar links a Resource with a mutex, to be held during all calls into the Avatar.

	```go
	type Avatar struct {
		// Has unexported fields.
	}
	```

- Free releases self to the ResPool.
If error is not nil, close it.

	```go
	func (avatar *Avatar) Free(err error)
	```

- ResPool returns ResPool to which it belongs.

	```go
	func (avatar *Avatar) ResPool() ResPool
	```

- ResPools stores ResPool.

	```go
	type ResPools struct {
		// Has unexported fields.
	}
	```

- NewResPools creates a new ResPools.

	```go
	func NewResPools() *ResPools
	```

- Clean delects and close all the ResPools.

	```go
	func (c *ResPools) Clean()
	```

- Del delects ResPool by name, and close the ResPool.

	```go
	func (c *ResPools) Del(name string)
	```

- Get gets ResPool by name.

	```go
	func (c *ResPools) Get(name string) (ResPool, bool)
	```

- GetAll gets all the ResPools.

	```go
	func (c *ResPools) GetAll() []ResPool
	```

- Set stores ResPool.
If the same name exists, will close and cover it.

	```go
	func (c *ResPools) Set(pool ResPool)
	```

### Shutdown

Close current program gracefully.

- import it

	```go
	"github.com/henrylee2cn/goutil/shutdown"
	```

- SetShutdown sets the function which is called after current program shutdown,
and the time-out period for current program shutdown.
If parameter timeout is 0, automatically use default `defaultTimeout`(60s).
If parameter timeout less than 0, it is indefinite period.
The finalizer function is executed before the shutdown deadline, but it is not guaranteed to be completed.

	```go
	func SetShutdown(timeout time.Duration, fn ...func() error)
	```

- Shutdown closes current program gracefully.
Parameter timeout is used to reset time-out period for current program shutdown.

	```go
	func Shutdown(timeout ...time.Duration)
	```
- Logger logger interface

	```go
	Logger interface {
		Infof(format string, v ...interface{})
		Errorf(format string, v ...interface{})
	}

- SetLog resets logger

	```go
	func SetLog(logger Logger)
	```
	```

### Various

Various small functions.

- import it

	```go
	"github.com/henrylee2cn/goutil/pool"
	```

- Byte2String convert []byte type to string type.

	```go
	func Bytes2String(b []byte) string
	```

- String2Bytes convert *string type to []byte type.
NOTE: panic if modify the member value of the []byte.

	```go
	func String2Bytes(s *string) []byte
	```

- RandomBytes returns securely generated random bytes. It will panic if the system's secure random number generator fails to function correctly.

	```go
	func RandomBytes(n int) []byte
	```

- RandomString returns a URL-safe, base64 encoded securely generated random string. It will panic if the system's secure random number generator fails to function correctly.
The length n must be an integer multiple of 4, otherwise the last character will be padded with `=`.

	```go
	func RandomString(n int) string
	```
