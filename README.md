- 👋 Hi, I’m @yaodongpo
- 👀 I’m interested in CS
- 🌱 I’m currently learning IOT
- 📫 How to reach me ：dongpouu@gmail.com

<!---
yaodongpo/yaodongpo is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->

package main

import (
	"context"
	"encoding/json"
	"io"
	"log"
	"sync"
	"time"
)

// Config 配置项
type Config struct {
	OriginURL     string        // 原始服
	"net/http"
	"os"务器地址
	ProxyAddr     string        // 代理服务监听地址
	CacheFile     string        // 缓存持久化文件路径
	CacheTTL      time.Duration // 缓存有效期
	OriginTimeout time.Duration // 回源请求超时时间
}

// CacheData 缓存数据结构
type CacheData struct {
	Data       json.RawMessage `json:"data"`
	UpdateTime time.Time       `json:"update_time"`
}

var (
	cfg = Config{
		OriginURL:     "http://localhost:8080/data",
		ProxyAddr:     ":8081",
		CacheFile:     "data_cache.json",
		CacheTTL:      5 * time.Minute,
		OriginTimeout: 10 * time.Second,
	}

	cache      *CacheData
	cacheMutex sync.RWMutex // 读写锁，保证并发安全
)

// loadCacheFromFile 从文件加载缓存
func loadCacheFromFile() (*CacheData, error) {
	file, err := os.Open(cfg.CacheFile)
	if err != nil {
		return nil, err
	}
	defer file.Close()

	var data CacheData
	if err := json.NewDecoder(file).Decode(&data); err != nil {
		return nil, err
	}
	return &data, nil
}

// saveCacheToFile 保存缓存到文件
func saveCacheToFile(data *CacheData) error {
	file, err := os.Create(cfg.CacheFile)
	if err != nil {
		return err
	}
	defer file.Close()

	return json.NewEncoder(file).Encode(data)
}

// fetchFromOrigin 从原始服务器拉取数据
func fetchFromOrigin() (json.RawMessage, error) {
	ctx, cancel := context.WithTimeout(context.Background(), cfg.OriginTimeout)
	defer cancel()

	req, err := http.NewRequestWithContext(ctx, "GET", cfg.OriginURL, nil)
	if err != nil {
		return nil, err
	}

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return nil, err
	}

	return io.ReadAll(resp.Body)
}

// getData 获取数据（优先缓存，否则回源）
func getData() (json.RawMessage, error) {
	// 1. 读缓存（读锁，高并发读安全）
	cacheMutex.RLock()
	if cache != nil && time.Since(cache.UpdateTime) < cfg.CacheTTL {
		defer cacheMutex.RUnlock()
		return cache.Data, nil
	}
	cacheMutex.RUnlock()

	// 2. 缓存失效/不存在，尝试回源（写锁，防止缓存击穿）
	cacheMutex.Lock()
	defer cacheMutex.Unlock()

	// 双重检查：防止多个 goroutine 同时进入后重复回源
	if cache != nil && time.Since(cache.UpdateTime) < cfg.CacheTTL {
		return cache.Data, nil
	}

	// 3. 回源拉取
	data, err := fetchFromOrigin()
	if err != nil {
		// 回源失败时，尝试返回旧缓存兜底
		if cache != nil {
			log.Printf("回源失败，返回旧缓存: %v", err)
			return cache.Data, nil
		}
		return nil, err
	}

	// 4. 更新内存缓存
	newCache := &CacheData{
		Data:       data,
		UpdateTime: time.Now(),
	}
	cache = newCache

	// 5. 异步持久化到文件（不阻塞主流程）
	go func() {
		if err := saveCacheToFile(newCache); err != nil {
			log.Printf("保存缓存到文件失败: %v", err)
		}
	}()

	return data, nil
}

// dataHandler 处理客户端请求
func dataHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodGet {
		http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
		return
	}

	data, err := getData()
	if err != nil {
		http.Error(w, "Failed to fetch data", http.StatusInternalServerError)
		log.Printf("获取数据失败: %v", err)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	w.Write(data)
}

func main() {
	// 启动时尝试加载历史缓存
	if data, err := loadCacheFromFile(); err == nil {
		cache = data
		log.Printf("成功加载历史缓存，最后更新时间: %s", data.UpdateTime.Format(time.RFC3339))
	} else {
		log.Printf("未找到历史缓存或加载失败: %v", err)
	}

	// 注册路由
	http.HandleFunc("/data", dataHandler)

	log.Printf("缓存代理服务启动，监听地址: %s", cfg.ProxyAddr)
	log.Printf("原始服务器地址: %s", cfg.OriginURL)
	log.Printf("缓存有效期: %s", cfg.CacheTTL)

	// 启动服务
	if err := http.ListenAndServe(cfg.ProxyAddr, nil); err != nil {
		log.Fatalf("服务启动失败: %v", err)
	}
}
