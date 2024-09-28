package main

import ( "context,fmt", "log", "net/http", "sync", "time")
 
)
// Cache interface for storing and retrieving data
type Cache interface {
 Set(key string, value interface{}, ttl time.Duration) error
 Get(key string) (interface{}, error)
 Delete(key string) error
}

// RedisCache implements the Cache interface using Redis
type RedisCache struct {
 client *redis.Client
}

// Set stores a value in the cache with a given TTL
func (rc *RedisCache) Set(key string, value interface{}, ttl time.Duration) error {
 return rc.client.Set(context.Background(), key, value, ttl).Err()
}

// Get retrieves a value from the cache
func (rc *RedisCache) Get(key string) (interface{}, error) {
 result, err := rc.client.Get(context.Background(), key).Result()
 if err == redis.Nil {
  return nil, fmt.Errorf("key not found: %s", key)
 } else if err != nil {
  return nil, err
 }
 return result, nil
}

// Delete removes a key from the cache
func (rc *RedisCache) Delete(key string) error {
 return rc.client.Del(context.Background(), key).Err()
}

// In-memory cache implementation
type InMemoryCache struct {
 data  map[string]interface{}
 mutex sync.RWMutex
}

// Set stores a value in the in-memory cache
func (imc *InMemoryCache) Set(key string, value interface{}, ttl time.Duration) error {
 imc.mutex.Lock()
 defer imc.mutex.Unlock()
 imc.data[key] = value
 return nil
}

// Get retrieves a value from the in-memory cache
func (imc *InMemoryCache) Get(key string) (interface{}, error) {
 imc.mutex.RLock()
 defer imc.mutex.RUnlock()
 value, ok := imc.data[key]
 if !ok {
  return nil, fmt.Errorf("key not found: %s", key)
 }
 return value, nil
}

// Delete removes a key from the in-memory cache
func (imc *InMemoryCache) Delete(key string) error {
 imc.mutex.Lock()
 defer imc.mutex.Unlock()
 delete(imc.data, key)
 return nil
}

// Function to handle requests to the cache
func handleCacheRequest(w http.ResponseWriter, r *http.Request) {
 key := r.URL.Path[1:]
 value, err := cache.Get(key)
 if err != nil {
  log.Println("Error retrieving from cache:", err)
  http.Error(w, "Error retrieving from cache", http.StatusInternalServerError)
  return
 }
 fmt.Fprintln(w, value)
}

var cache Cache

func main() {
 // Initialize Redis client
 redisClient := redis.NewClient(&redis.Options{
  Addr:     "localhost:6379",
  Password: "", // No password set for this example
  DB:0,  // Use default database
 }
 // Check Redis connection
 if _, err := redisClient.Ping(context.Background()).Result(); err != nil {
  log.Fatal("Error connecting to Redis:", err)
 }

 // Use Redis as the cache
 cache = &RedisCache{client: redisClient}

 // Initialize in-memory cache
 // cache = &InMemoryCache{data: make(map[string]interface{})}

 // Handle cache requests
 http.HandleFunc("/" + "[:a-zA-Z0-9]+", handleCacheRequest)

 // Start the server
 fmt.Println("Server listening on port 8080")
 log.Fatal(http.ListenAndServe(":8080", nil))
}
