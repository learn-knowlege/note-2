# 限流

### 常用方式

> 限流的常用处理手段有：计数器、滑动窗口、漏桶、令牌桶。

### 计数器

> 计数器是一种比较简单的限流算法，选定一个时间窗口，每当有接口请求到来，我们就将计数器加一。
> 累加访问次数超过限流值的情况时，我们就拒绝后续的访问请求。当进入下一个时间窗口之后，计数器就清零重新计数。

<img src="image/hit.jpg" width=500>

> 假设我们的限流规则是，每秒钟不能超过 100 次接口请求。第一个 1s 时间窗口内，100 次接口请求都集中在最后 
> 10ms 内。在第二个 1s 的时间窗口内，100 次接口请求都集中在最开始的 10ms 内。虽然两个时间窗口内流量都符
> 合限流要求（≤100 个请求），但在两个时间窗口临界的 20ms 内，会集中有 200 次接口请求。固定时间窗口限流算法
> 并不能对这种情况做限制，所以，集中在这 20ms 内的 200 次请求就有可能压垮系统。

<img src="image/hit2.jpg" width=500>

### 滑动窗口

> 针对计数器算法进行改造，滑动窗口从流量曲线上来看会更加平滑。

> 限流的规则是，假设在任意 1s 内，接口的请求次数都不能大于 K 次。我们就维护一个大小为 K+1 的循环队列。

> 当有新的请求到来时，我们将与这个新请求的时间间隔超过 1s 的请求，从队列中删除。然后，我们再来看循环队列中
> 是否有空闲位置。如果有，则把新请求存储在队列尾部（tail 指针所指的位置）；如果没有，则说明这 1 秒内的请求
> 次数已经超过了限流值 K，所以这个请求被拒绝服务。

<img src="image/sliding.jpg" width=480>

> 即便滑动时间窗口限流算法可以保证任意时间窗口内，接口请求次数都不会超过最大限流值，但是仍然不能防止，在细时间粒度上访问过于集中的问题。
> 比如，第一个 1s 的时间窗口内，100 次请求都集中在最后 10ms 中基于时间窗口的限流算法，不管是固定时间窗口还是滑动时间窗口，只能在选定
> 的时间粒度上限流，对选定时间粒度内的更加细粒度的访问频率不做限制。


### 漏桶算法

> 漏桶算法(Leaky Bucket)它的主要目的是控制数据注入到网络的速率，平滑网络上的突发流量。
> 漏桶算法提供了一种机制，通过它，突发流量可以被整形以便为网络提供一个稳定的流量。

> 漏桶可以看作是一个带有常量服务时间的单服务器队列，如果漏桶（包缓存）溢出，那么数据包会被丢弃。 在网络中，漏桶
> 算法可以控制端口的流量输出速率，平滑网络上的突发流量，实现流量整形，从而为网络提供一个稳定的流量。

> 如图所示，把请求比作是水，水来了都先放进桶里，并以限定的速度出水，当水来得过猛而出水不够快时就会导致水直接溢出，
> 即拒绝服务。

<img src="image/bucket.png" width=400>

> 漏桶算法能强行限制数据的传输速率。

```cgo
type Bucket struct {
	// 容量
	capacity uint64
	// 水位
	water uint64
	// 纳秒
	reset uint64
	// 每1秒10滴  如果想做更细粒度的时间可以把单位换成毫秒
	rate uint64
}

func (b *Bucket) Acquire() bool {
	// 流出
	left := ((unixNano() - b.reset) / uint64(time.Second)) * b.rate
	if left > b.water {
		b.water = 0
	} else {
		b.water -= left
	}
	b.reset = unixNano()
	if b.water < b.capacity {
		b.water++
		return true
	} else {
		return false
	}
}

func NewBucket(capacity uint64, rate uint64) *Bucket {
	return &Bucket{
		capacity: capacity,
		water:    0,
		reset:    unixNano(),
		rate:     rate,
	}
}

// unixNano 当前时间（纳秒） 1秒=1000000000纳秒
func unixNano() uint64 {
	return uint64(time.Now().UnixNano())
}


func main() {
	bucket := NewBucket(100, 10)
	for i := 0; i < 102; i++ {
		fmt.Println(bucket.Acquire())
	}
	time.Sleep(time.Second)
	for i := 0; i < 12; i++ {
		fmt.Println(bucket.Acquire())
	}
}
```

### 令牌桶

> 令牌桶算法用来控制发送到网络上的数据的数目，并允许突发数据的发送。

> 令牌桶算法的原理是系统会以一个恒定的速度往桶里放入令牌，而如果请求需要被处理，则需要先从桶里获取一个令牌，

> 当桶里没有令牌可取时，则拒绝服务。从原理上看，令牌桶算法和漏桶算法是相反的，一个“进水”，一个是“漏水”。

<img src="image/limit.jpg" width=400>

```cgo
type RateLimiter struct {
	rate      uint64
	allowance uint64
	max       uint64
	unit      uint64
	lastCheck uint64
}

// Limit 判断是否超过限制
func (rl *RateLimiter) Limit() bool {
	//fmt.Println("limit")
	now := unixNano()
	// 计算上一次调用到现在过了多少纳秒
	passed := now - atomic.SwapUint64(&rl.lastCheck, now)
	rate := atomic.LoadUint64(&rl.rate)
	current := atomic.AddUint64(&rl.allowance, passed*rate)
	if max := atomic.LoadUint64(&rl.max); current > max {
		atomic.AddUint64(&rl.allowance, max-current)
		current = max
	}
	if current < rl.unit {
		return true
	}
	// 没有超过限额
	atomic.AddUint64(&rl.allowance, -rl.unit)
	return false
}

// New 创建RateLimiter实例
func NewRateLimiter(rate int, per time.Duration) *RateLimiter {
	// 纳秒   time.Second = 1000000000
	nano := uint64(per)
	if nano < 1 {
		nano = uint64(time.Second)
	}
	if rate < 1 {
		rate = 1
	}
	return &RateLimiter{
		rate:      uint64(rate),
		allowance: uint64(rate) * nano,
		max:       uint64(rate) * nano,
		unit:      nano,
		lastCheck: unixNano(),
	}
}

// unixNano 当前时间（纳秒） 1秒=1000000000纳秒
func unixNano() uint64 {
	return uint64(time.Now().UnixNano())
}

func main() {
	rateLimiter := NewRateLimiter(10, time.Second)
	for i := 0; i < 102; i++ {
		fmt.Println(i,rateLimiter.Limit())
	}
	time.Sleep(time.Second)
	for i := 0; i < 12; i++ {
		fmt.Println(i,rateLimiter.Limit())
	}
}
```

### 总结

> 漏桶算法和令牌桶算法的选择

> 漏桶算法与令牌桶算法在表面看起来类似，很容易将两者混淆。但事实上，这两者具有截然不同的特性，且为不同的目的而使用。

> 漏桶算法与令牌桶算法的区别在于，漏桶算法能够强行限制数据的传输速率，令牌桶算法能够在限制数据的平均传输速率的同时还允许某种程度的突发传输。

> 需要注意的是，在某些情况下，漏桶算法不能够有效地使用网络资源，因为漏桶的漏出速率是固定的，所以即使网络中没有发生拥塞，漏桶算法也不能使某一个单独的数据流达到端口速率。因此，漏桶算法对于存在突发特性的流量来说缺乏效率。而令牌桶算法则能够满足这些具有突发特性的流量。通常，漏桶算法与令牌桶算法结合起来为网络流量提供更高效的控制。


