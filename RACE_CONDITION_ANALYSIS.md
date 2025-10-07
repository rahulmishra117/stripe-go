# Complex Race Condition in GetBackend Function

## Location
`stripe.go`, lines 1220-1252, in the `GetBackend` function

## Issue Description
There is a **check-then-act race condition** in the lazy initialization of backend instances. Multiple goroutines can simultaneously create duplicate backend instances when accessing a backend for the first time.

## The Vulnerable Code

```go
func GetBackend(backendType SupportedBackend) Backend {
	var backend Backend

	backends.mu.RLock()
	switch backendType {
	case APIBackend:
		backend = backends.API
	case ConnectBackend:
		backend = backends.Connect
	case UploadsBackend:
		backend = backends.Uploads
	case MeterEventsBackend:
		backend = backends.MeterEvents
	}
	backends.mu.RUnlock()
	if backend != nil {
		return backend
	}

	// RACE CONDITION: Multiple goroutines can reach here simultaneously
	backend = GetBackendWithConfig(
		backendType,
		&BackendConfig{
			HTTPClient:        httpClient,
			LeveledLogger:     nil,
			MaxNetworkRetries: nil,
			URL:               nil,
		},
	)

	SetBackend(backendType, backend)

	return backend
}
```

## Race Condition Scenario

1. **Thread A** calls `GetBackend(APIBackend)`, acquires read lock
2. **Thread B** calls `GetBackend(APIBackend)`, acquires read lock (readers don't block each other)
3. Both threads find `backends.API == nil`
4. Both threads release their read locks
5. **Thread A** calls `GetBackendWithConfig` and creates Backend Instance #1
6. **Thread B** calls `GetBackendWithConfig` and creates Backend Instance #2 
7. **Thread A** calls `SetBackend` and stores Instance #1
8. **Thread B** calls `SetBackend` and stores Instance #2 (overwrites Instance #1)
9. **Thread A** returns Instance #1 to its caller
10. **Thread B** returns Instance #2 to its caller

## Impact

### 1. **Inconsistent Backend State**
Different parts of the application may be using different backend instances, leading to:
- Inconsistent telemetry tracking
- Different HTTP clients with potentially different connection pools
- Different retry configurations

### 2. **Resource Leaks**
- Multiple HTTP clients are created unnecessarily
- Connection pools are duplicated
- Memory is wasted on unused backend instances

### 3. **Performance Degradation**
- Unnecessary backend creation overhead
- Multiple HTTP client connection pools instead of reusing one
- Reduced connection pooling efficiency

### 4. **Subtle Bugs**
- Race detector (`go test -race`) would likely catch this
- Could lead to hard-to-reproduce bugs in multi-threaded environments
- Different goroutines might see different backend configurations

## Proof of Concept

```go
// This test would demonstrate the race condition
func TestGetBackendRaceCondition(t *testing.T) {
    // Reset backends
    backends = Backends{}
    
    var wg sync.WaitGroup
    backends := make([]Backend, 10)
    
    // Launch 10 goroutines simultaneously
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(idx int) {
            defer wg.Done()
            backends[idx] = GetBackend(APIBackend)
        }(i)
    }
    
    wg.Wait()
    
    // Check if all goroutines got the same backend instance
    firstBackend := backends[0]
    for i := 1; i < 10; i++ {
        if backends[i] != firstBackend {
            t.Errorf("Race condition detected: backend %d is different from backend 0", i)
        }
    }
}
```

## Recommended Fix

Use **sync.Once** for each backend type to ensure thread-safe lazy initialization:

```go
type Backends struct {
	API, Connect, Uploads, MeterEvents Backend
	mu                                 sync.RWMutex
	apiOnce, connectOnce, uploadsOnce, meterEventsOnce sync.Once
}

func GetBackend(backendType SupportedBackend) Backend {
	var backend Backend

	// Try to get existing backend
	backends.mu.RLock()
	switch backendType {
	case APIBackend:
		backend = backends.API
	case ConnectBackend:
		backend = backends.Connect
	case UploadsBackend:
		backend = backends.Uploads
	case MeterEventsBackend:
		backend = backends.MeterEvents
	}
	backends.mu.RUnlock()
	
	if backend != nil {
		return backend
	}

	// Use sync.Once to ensure only one goroutine initializes each backend
	switch backendType {
	case APIBackend:
		backends.apiOnce.Do(func() {
			if backends.API == nil {  // Double-check inside Once
				b := GetBackendWithConfig(backendType, &BackendConfig{
					HTTPClient:        httpClient,
					LeveledLogger:     nil,
					MaxNetworkRetries: nil,
					URL:               nil,
				})
				SetBackend(backendType, b)
			}
		})
		return backends.API
	case ConnectBackend:
		backends.connectOnce.Do(func() {
			if backends.Connect == nil {
				b := GetBackendWithConfig(backendType, &BackendConfig{
					HTTPClient:        httpClient,
					LeveledLogger:     nil,
					MaxNetworkRetries: nil,
					URL:               nil,
				})
				SetBackend(backendType, b)
			}
		})
		return backends.Connect
	// ... similar for other backends
	}

	return backend
}
```

## Severity
**Medium to High** - While this may not cause crashes, it can lead to:
- Resource leaks in high-concurrency environments
- Subtle bugs that are difficult to diagnose
- Performance degradation
- Inconsistent behavior across goroutines

This is particularly problematic in applications that:
- Initialize the Stripe client from multiple goroutines
- Use the global backend pattern (not the recommended `stripe.Client` pattern)
- Have high concurrency requirements
