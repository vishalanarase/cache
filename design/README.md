# LRU cache in Go

✅ Per-key TTL  
✅ Background cleanup of expired entries  
✅ Eviction when max capacity is exceeded (Least Recently Used item is removed)  

---

### 📦 `lru_cache.go`

```go
package cache

import (
	"container/list"
	"sync"
	"time"
)

type CacheItem[T any] struct {
	Key        string
	Value      T
	Expiration int64 // UnixNano timestamp
}

type LRUCache[T any] struct {
	capacity     int
	items        map[string]*list.Element
	evictionList *list.List
	mutex        sync.RWMutex
	stopCleaner  chan struct{}
}

func NewLRUCache[T any](capacity int, cleanupInterval time.Duration) *LRUCache[T] {
	cache := &LRUCache[T]{
		capacity:     capacity,
		items:        make(map[string]*list.Element),
		evictionList: list.New(),
		stopCleaner:  make(chan struct{}),
	}
	go cache.startCleanup(cleanupInterval)
	return cache
}

func (c *LRUCache[T]) Set(key string, value T, ttl ...time.Duration) {
	c.mutex.Lock()
	defer c.mutex.Unlock()

	if ele, found := c.items[key]; found {
		c.evictionList.Remove(ele)
	}

	var expireAt int64
	if len(ttl) > 0 && ttl[0] > 0 {
		expireAt = time.Now().Add(ttl[0]).UnixNano()
	}

	item := CacheItem[T]{Key: key, Value: value, Expiration: expireAt}
	entry := c.evictionList.PushFront(item)
	c.items[key] = entry

	if len(c.items) > c.capacity {
		c.evictOldest()
	}
}

func (c *LRUCache[T]) Get(key string) (T, bool) {
	c.mutex.Lock()
	defer c.mutex.Unlock()

	var zero T
	entry, found := c.items[key]
	if !found {
		return zero, false
	}

	item := entry.Value.(CacheItem[T])
	if item.Expiration > 0 && time.Now().UnixNano() > item.Expiration {
		c.evictionList.Remove(entry)
		delete(c.items, key)
		return zero, false
	}

	c.evictionList.MoveToFront(entry)
	return item.Value, true
}

func (c *LRUCache[T]) Delete(key string) {
	c.mutex.Lock()
	defer c.mutex.Unlock()

	if entry, found := c.items[key]; found {
		c.evictionList.Remove(entry)
		delete(c.items, key)
	}
}

func (c *LRUCache[T]) evictOldest() {
	oldest := c.evictionList.Back()
	if oldest != nil {
		item := oldest.Value.(CacheItem[T])
		delete(c.items, item.Key)
		c.evictionList.Remove(oldest)
	}
}

func (c *LRUCache[T]) cleanupExpired() {
	now := time.Now().UnixNano()
	for key, entry := range c.items {
		item := entry.Value.(CacheItem[T])
		if item.Expiration > 0 && now > item.Expiration {
			c.evictionList.Remove(entry)
			delete(c.items, key)
		}
	}
}

func (c *LRUCache[T]) startCleanup(interval time.Duration) {
	ticker := time.NewTicker(interval)
	defer ticker.Stop()

	for {
		select {
		case <-ticker.C:
			c.mutex.Lock()
			c.cleanupExpired()
			c.mutex.Unlock()
		case <-c.stopCleaner:
			return
		}
	}
}

func (c *LRUCache[T]) Stop() {
	close(c.stopCleaner)
}
```

---

### 🧪 Example Usage

```go
package main

import (
	"fmt"
	"time"

	"your-module-name/cache"
)

func main() {
	cache := cache.NewLRUCache[string](2, 30*time.Second)
	defer cache.Stop()

	cache.Set("a", "apple", 3*time.Second)
	cache.Set("b", "banana")
	cache.Set("c", "cherry") // will evict "a" if expired or "b" if "a" not expired

	val, ok := cache.Get("a")
	fmt.Println("a exists?", ok)

	val, ok = cache.Get("b")
	fmt.Println("b:", val, ok)

	val, ok = cache.Get("c")
	fmt.Println("c:", val, ok)
}
```

---