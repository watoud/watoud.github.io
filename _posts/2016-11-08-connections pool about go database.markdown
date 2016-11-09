---
layout: post
title: Go数据库连接池浅析
comments: true
archive: true
tag: [GO, database, connection pool]
---
Go数据库框架提供了连接池的管理，在需要跟数据库建立连接时，先从连接池中查找是否存在可用的闲置连接，不存在的情况下才创建新的连接。为了方便对连接池进行管理，框架提供了一些参数，比如允许打开的最大连接数、最大闲置连接数等。

## 连接的创建
在`sql.DB`结构体中有一个字段`freeConn []*driverConn`，数据库中已经创建的闲置连接就保存在这个Slice中。在数据库操作中，使用类似`db, err := sql.Open("mysql", "watoud:watoud@/testdb")`语句打开数据库的时候，并不会去连接数据库，只有在执行具体操作需要的时候才会去创建连接。

~~~~~go
func Open(driverName, dataSourceName string) (*DB, error) {
	driversMu.RLock()
	// 检查数据库驱动是否存在
	driveri, ok := drivers[driverName]
	driversMu.RUnlock()
	if !ok {
		return nil, fmt.Errorf("sql: unknown driver %q (forgotten import?)", driverName)
	}
	// 构建DB对象
	db := &DB{
		driver:   driveri,
		dsn:      dataSourceName,
		openerCh: make(chan struct{}, connectionRequestQueueSize),
		lastPut:  make(map[*driverConn]string),
	}
	// 启动一个goroutine在channel收到消息时创建连接
	go db.connectionOpener()
	return db, nil
}

func (db *DB) connectionOpener() {
	for range db.openerCh {
		// 创建新的连接
		db.openNewConnection()
	}
}
~~~~~

从上面的代码中可以看出，DB.Open方法主要做了三件事。

- 根据数据库驱动名称检查驱动是否已经注册
- 初始化DB对象
- 启动一个goroutine不停地在channel上等待接收数据，一旦接收到就创建新的连接

那么具体什么时候数据库才去创建连接呢？不妨看一下DB.Query的详细调用过程。

~~~~~go
- func (db *DB) Query(query string, args ...interface{}) (*Rows, error) 
	- func (db *DB) query(query string, args []interface{}, strategy connReuseStrategy) (*Rows, error)
		- func (db *DB) conn(strategy connReuseStrategy) (*driverConn, error)
~~~~~

数据库在执行具体的操作前会先去获取连接，接着看获取连接的详细操作过程。

~~~~~go
func (db *DB) conn(strategy connReuseStrategy) (*driverConn, error) {
	db.mu.Lock()
	if db.closed {
		db.mu.Unlock()
		return nil, errDBClosed
	}
	lifetime := db.maxLifetime

	// Prefer a free connection, if possible.
	numFree := len(db.freeConn)

	// 如果存在可用的闲置连接则直接重用闲置的连接
	if strategy == cachedOrNewConn && numFree > 0 {
		conn := db.freeConn[0]
		copy(db.freeConn, db.freeConn[1:])
		db.freeConn = db.freeConn[:numFree-1]
		conn.inUse = true
		db.mu.Unlock()
		if conn.expired(lifetime) {
			conn.Close()
			return nil, driver.ErrBadConn
		}
		return conn, nil
	}

	// 如果没有闲置的连接并且已经打开的连接已经达到允许创建的最大连接数，
	// 那么该操作将会等待，要么超时，要么从channel中接收到一个连接
	if db.maxOpen > 0 && db.numOpen >= db.maxOpen {
		// Make the connRequest channel. It's buffered so that the
		// connectionOpener doesn't block while waiting for the req to be read.
		req := make(chan connRequest, 1)
		db.connRequests = append(db.connRequests, req)
		db.mu.Unlock()
		ret, ok := <-req
		if !ok {
			return nil, errDBClosed
		}
		if ret.err == nil && ret.conn.expired(lifetime) {
			ret.conn.Close()
			return nil, driver.ErrBadConn
		}
		return ret.conn, ret.err
	}

	// 这个时候既没有可用的闲置连接，又没有超过最大连接上限，则创建新的连接
	db.numOpen++
	db.mu.Unlock()
	ci, err := db.driver.Open(db.dsn)
	if err != nil {
		db.mu.Lock()
		db.numOpen-- // correct for earlier optimism
		db.maybeOpenNewConnections()
		db.mu.Unlock()
		return nil, err
	}
	db.mu.Lock()
	dc := &driverConn{
		db:        db,
		createdAt: nowFunc(),
		ci:        ci,
	}
	db.addDepLocked(dc, dc)
	dc.inUse = true
	db.mu.Unlock()
	return dc, nil
}
~~~~~

从上面的代码中可以看出，当执行一个数据库操作需要连接数据库的时候，首先检查是否有可用的闲置连接，如果有的话就重用闲置的连接。如果没有闲置的可用连接，则检查已经打开的连接个数是否达到允许的最大连接数，如果是的话就开始等待，没有达到上限的情况下才会创建新的连接。

## 连接池的管理
Go数据库框架提供了三个参数方便对连接池进行管理，分别是最大闲置连接数、允许打开的最大连接数以及连接的最长存在时间，接下来看下这三个参数连接池的影响。

### 首先看下参数连接最长存在时间MaxLifetime

~~~~~go
func (db *DB) SetConnMaxLifetime(d time.Duration) {
	if d < 0 {
		d = 0
	}
	db.mu.Lock()
	// wake cleaner up when lifetime is shortened.
	if d > 0 && d < db.maxLifetime && db.cleanerCh != nil {
		select {
		// 向cleanerch中发送数据，需要注意的是，cleanerch的缓存大小为1
		// 如果这个chennel里面还有数据没有来得及处理，那么这个消息就不会向chan里面发送了
		// 所以才有了在获取闲置连接时判断连接是否超时的处理
		case db.cleanerCh <- struct{}{}:
		default:
		}
	}
	db.maxLifetime = d
	db.startCleanerLocked()
	db.mu.Unlock()
}

func (db *DB) startCleanerLocked() {
	// maxLifetime大于0，已经有打开的连接，
	//且在channel没有创建的情况下才会启动一个goroutine
	if db.maxLifetime > 0 && db.numOpen > 0 && db.cleanerCh == nil {
		db.cleanerCh = make(chan struct{}, 1)
		go db.connectionCleaner(db.maxLifetime)
	}
}

func (db *DB) connectionCleaner(d time.Duration) {
	const minInterval = time.Second

	if d < minInterval {
		// 定时周期至少为1秒
		d = minInterval
	}
	// 以最长存在时间设置定时器
	t := time.NewTimer(d)

	for {
		select {
		case <-t.C:  // 定时器超时
		case <-db.cleanerCh: // 设置了超时时间或者关闭数据库的时候会向cleanerCh发送数据
		}

		db.mu.Lock()
		d = db.maxLifetime
		if db.closed || db.numOpen == 0 || d <= 0 {
			// 数据库已经关闭或者没有打开的连接或者最长超时时间无限长时退出该goroutine
			db.cleanerCh = nil
			db.mu.Unlock()
			return
		}

		expiredSince := nowFunc().Add(-d)
		var closing []*driverConn
		for i := 0; i < len(db.freeConn); i++ {
			c := db.freeConn[i]
			if c.createdAt.Before(expiredSince) {
				// 把超时的连接加入到待关闭的slice中
				closing = append(closing, c)
				last := len(db.freeConn) - 1
				db.freeConn[i] = db.freeConn[last]
				db.freeConn[last] = nil
				db.freeConn = db.freeConn[:last]
				i--
			}
		}
		db.mu.Unlock()
		// 关闭超过最大存在时长的连接
		for _, c := range closing {
			c.Close()
		}

		if d < minInterval {
			d = minInterval
		}
		// 重置定时器
		t.Reset(d)
	}
}
~~~~~

从上面可以看出，调用接口`DB.SetConnMaxLifetime`给数据库设置连接最长存在时间时，会启动一个goroutine去循环检测连接池中的闲置连接是否超时并关闭超时的连接。当打开的连接数为0时会退出循环检测，这可以避免不要的检测操作，但是当创建了新的连接时，在什么地方会再次启动这个循环检测的操作呢？

~~~go
- func (db *DB) Exec(query string, args ...interface{}) (Result, error)
	- func (db *DB) exec(query string, args []interface{}, strategy connReuseStrategy) 
		- func (db *DB) putConn(dc *driverConn, err error)
			- func (db *DB) putConnDBLocked(dc *driverConn, err error) bool
				- func (db *DB) startCleanerLocked()
					- func (db *DB) connectionCleaner(d time.Duration)
~~~

从上面的调用示例中不难看出，当没有打开的连接时，如果执行某个具体的数据库操作，这时需要创建一个连接，而等待这个操作执行完成之后，把这个闲置连接放到连接池中时会去尝试启动一个goroutine去循环检测闲置的连接是否超过了最长存在时长。

### 接下来看下最大闲置连接数

~~~~~go
func (db *DB) SetMaxIdleConns(n int) {
	db.mu.Lock()
	if n > 0 {
		db.maxIdle = n
	} else {
		// No idle connections.
		db.maxIdle = -1
	}
	// Make sure maxIdle doesn't exceed maxOpen
	if db.maxOpen > 0 && db.maxIdleConnsLocked() > db.maxOpen {
		db.maxIdle = db.maxOpen
	}
	var closing []*driverConn
	idleCount := len(db.freeConn)
	maxIdle := db.maxIdleConnsLocked()
	if idleCount > maxIdle {
		closing = db.freeConn[maxIdle:]
		db.freeConn = db.freeConn[:maxIdle]
	}
	db.mu.Unlock()
	for _, c := range closing {
		c.Close()
	}
}
~~~~~

在设置最大闲置连接数的时候，会立即检查一下当前闲置的连接是否超出了设置的上限，并把超过上限的多余的连接关闭掉。但是从这个函数中并没有看到有启动goroutine去周期性的检查闲置的连接是否超过了设置的上限，那这个又是在哪里处理的呢？

~~~~~go
- func (db *DB) Exec(query string, args ...interface{}) (Result, error)
	- func (db *DB) exec(query string, args []interface{}, strategy connReuseStrategy) 
		- func (db *DB) putConn(dc *driverConn, err error)
			- func (db *DB) putConnDBLocked(dc *driverConn, err error) bool

// ---------------------------------------------------------------------

func (db *DB) putConn(dc *driverConn, err error) {
	// ...... 其他代码
	added := db.putConnDBLocked(dc, nil)
	// ...... 其他代码

	if !added {
		dc.Close()
	}
}

func (db *DB) putConnDBLocked(dc *driverConn, err error) bool {
	// 打开的连接数已经大于允许打开的最大连接数时立即返回false
	if db.maxOpen > 0 && db.numOpen > db.maxOpen {
		return false
	}
	if c := len(db.connRequests); c > 0 {
		// 有操作因为当前没有可用的闲置连接，并且打开的连接也已经达到了允许的最大值
		// 这个操作就会创建一个connRequest的chennle并等待接收一个连接
		// 在这种情况下，会优先处理
		req := db.connRequests[0]
		copy(db.connRequests, db.connRequests[1:])
		db.connRequests = db.connRequests[:c-1]
		if err == nil {
			dc.inUse = true
		}
		req <- connRequest{
			conn: dc,
			err:  err,
		}
		return true
	} else if err == nil && !db.closed && db.maxIdleConnsLocked() > len(db.freeConn) {
		// 在把当前连接加入到闲置的连接池中时，会先判断已经闲置的连接是否已经达到允许的最大
		// 上限，在没有达到上限的情况下才会加入到闲置的连接池中
		db.freeConn = append(db.freeConn, dc)
		db.startCleanerLocked()
		return true
	}
	return false
}
~~~~~

一个连接在被使用完成后准备把他放到连接池中去之前都会调用到`DB.putConn`,该方法会调用到`DB.putConnDBLocked`方法并根据这个方法的返回值来确定是否关键这个连接。在`DB.putConnDBLocked`方法中可以看到，当已经闲置的连接数已经达到上限时会返回false，从而`DB.putConn`会关闭掉这个连接。

### 最后看下最大打开连接数

~~~~~go
func (db *DB) SetMaxOpenConns(n int) {
	db.mu.Lock()
	db.maxOpen = n
	if n < 0 {
		db.maxOpen = 0
	}
	// 如果最大的闲置连接数比允许打开的最大连接数的话，会立即调用
	// SetMaxIdleConns关闭超出的闲置连接
	syncMaxIdle := db.maxOpen > 0 && db.maxIdleConnsLocked() > db.maxOpen
	db.mu.Unlock()
	if syncMaxIdle {
		db.SetMaxIdleConns(n)
	}
}
~~~~~

上面的方法主要就是设置了一下允许打开的最大连接数，并在最大闲置连接数大于允许打开的最大连接数时进行一些相关处理。接下来看下，在具体需要创建新连接的时候会怎么处理。

~~~~~go
- func (db *DB) Exec(query string, args ...interface{}) (Result, error)
	- func (db *DB) exec(query string, args []interface{}, strategy connReuseStrategy) (res Result, err error)
		- func (db *DB) conn(strategy connReuseStrategy) (*driverConn, error)

// ----------------------------------------------------------------------------------

func (db *DB) conn(strategy connReuseStrategy) (*driverConn, error) {
	db.mu.Lock()
	if db.closed {
		db.mu.Unlock()
		return nil, errDBClosed
	}
	lifetime := db.maxLifetime

	numFree := len(db.freeConn)
	if strategy == cachedOrNewConn && numFree > 0 {
		// 如果有限制的连接，则直接重用闲置的连接
		conn := db.freeConn[0]
		copy(db.freeConn, db.freeConn[1:])
		db.freeConn = db.freeConn[:numFree-1]
		conn.inUse = true
		db.mu.Unlock()
		if conn.expired(lifetime) {
			conn.Close()
			return nil, driver.ErrBadConn
		}
		return conn, nil
	}

	if db.maxOpen > 0 && db.numOpen >= db.maxOpen {
		// 当没有闲置的连接并且打开的连接已经达到允许的最大上限时，
		// 会在数据库的等待连接的channel slice中新追加一个，
		// 并等待从channel上面接收一个connRequest，该对象中包含一个连接信息
		req := make(chan connRequest, 1)
		db.connRequests = append(db.connRequests, req)
		db.mu.Unlock()
		ret, ok := <-req
		if !ok {
			return nil, errDBClosed
		}
		if ret.err == nil && ret.conn.expired(lifetime) {
			ret.conn.Close()
			return nil, driver.ErrBadConn
		}
		return ret.conn, ret.err
	}

	db.numOpen++ // optimistically
	db.mu.Unlock()
	// 当没有闲置的连接并且打开的连接没有达到允许的最大上限时
	// 这个时候才会直接创建新的连接
	ci, err := db.driver.Open(db.dsn)
	
	// ...... 其他代码
}
~~~~~

当一个数据库操作因为打开的连接数已经达到上限而在channel上等待时，那么什么情况下会有其他操作向这个channel上面发送connRequest对象呢？

~~~~~go
- func (dc *driverConn) releaseConn(err error)
	- func (db *DB) putConn(dc *driverConn, err error)
		- func (db *DB) putConnDBLocked(dc *driverConn, err error) bool 

// ------------------------------------------------------------------------

func (db *DB) putConnDBLocked(dc *driverConn, err error) bool {
	if db.maxOpen > 0 && db.numOpen > db.maxOpen {
		return false
	}
	if c := len(db.connRequests); c > 0 {
		req := db.connRequests[0]
		copy(db.connRequests, db.connRequests[1:])
		db.connRequests = db.connRequests[:c-1]
		if err == nil {
			dc.inUse = true
		}
		req <- connRequest{
			conn: dc,
			err:  err,
		}
		return true
	} else if err == nil && !db.closed && db.maxIdleConnsLocked() > len(db.freeConn) {
		db.freeConn = append(db.freeConn, dc)
		db.startCleanerLocked()
		return true
	}
	return false
}
~~~~~

从上面可以看出，当执行完一个操作去释放连接的时候，系统会去检查是否有正在`db.connRequests`上等待连接的操作，如果有就会向这个channel中发送一个connRequest对象，从而正在等待的操作就可以获得连接了。





