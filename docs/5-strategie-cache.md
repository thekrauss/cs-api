# Stratégie de Cache (version courte)

## 1) Interface utilisée

```go
type AuthCache interface {
    Get(ctx context.Context, key string) (string, error)
    Set(ctx context.Context, key string, value string, ttl time.Duration) error
    Delete(ctx context.Context, key string) error
}
```

## 2) Factory depuis la config

```go
func NewAuthCacheFromConfig(ctx context.Context, cfg config.RedisConfig) AuthCache {
    ristrettoCache, err := NewRistrettoAuthCache()
    if err != nil {
        log.Fatalf("failed to initialize ristretto cache: %v", err)
    }

    if !cfg.Enabled {
        return ristrettoCache
    }

    addr := fmt.Sprintf("%s:%d", cfg.Host, cfg.Port)
    redisCache, err := NewRedisAuthCache(ctx, addr, cfg.Password, cfg.DB)
    if err != nil {
        log.Printf("redis enabled but connection failed, fallback to ristretto: %v", err)
        return ristrettoCache
    }

    return NewFallbackAuthCache(redisCache, ristrettoCache)
}
```

## 3) Fallback (Redis -> Ristretto)

```go
type FallbackAuthCache struct {
    primary  AuthCache
    fallback AuthCache
}

func NewFallbackAuthCache(primary AuthCache, fallback AuthCache) *FallbackAuthCache {
    return &FallbackAuthCache{primary: primary, fallback: fallback}
}

func (c *FallbackAuthCache) Get(ctx context.Context, key string) (string, error) {
    val, err := c.primary.Get(ctx, key)
    if err != nil || val == "" {
        return c.fallback.Get(ctx, key)
    }
    return val, nil
}

func (c *FallbackAuthCache) Set(ctx context.Context, key string, value string, ttl time.Duration) error {
    if err := c.primary.Set(ctx, key, value, ttl); err != nil {
        return c.fallback.Set(ctx, key, value, ttl)
    }
    return nil
}
```

## 4) Implémentation Ristretto (L1)

```go
type RistrettoAuthCache struct {
	cache *ristretto.Cache
}

func NewRistrettoAuthCache() (*RistrettoAuthCache, error) {
    cache, err := ristretto.NewCache(&ristretto.Config{
        NumCounters: 1e7,
        MaxCost:     1 << 30,
        BufferItems: 64,
    })
    if err != nil {
        return nil, err
    }
    return &RistrettoAuthCache{cache: cache}, nil
}

func (c *RistrettoAuthCache) Get(ctx context.Context, key string) (string, error) {
    val, ok := c.cache.Get("gen:" + key)
    if !ok {
        return "", nil
    }
    s, ok := val.(string)
    if !ok {
        return "", fmt.Errorf("value is not a string")
    }
    return s, nil
}
```

## 5) Implémentation Redis (L2)

```go

type RedisAuthCache struct {
	client *redis.Client
}

func NewRedisAuthCache(ctx context.Context, addr string, password string, db int) (*RedisAuthCache, error) {
    if addr == "" {
        return nil, fmt.Errorf("redis address is required")
    }

    client := redis.NewClient(&redis.Options{Addr: addr, Password: password, DB: db})
    if err := client.Ping(ctx).Err(); err != nil {
        return nil, err
    }

    return &RedisAuthCache{client: client}, nil
}

func (c *RedisAuthCache) Get(ctx context.Context, key string) (string, error) {
    val, err := c.client.Get(ctx, key).Result()
    if err == redis.Nil {
        return "", nil
    }
    return val, err
}
```
