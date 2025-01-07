# 前端 API 调用

## 1. Axios 使用

### 1.1 基本配置
```js
import axios from 'axios';

// 创建实例
const api = axios.create({
  baseURL: 'https://api.example.com',
  timeout: 5000,
  headers: {
    'Content-Type': 'application/json'
  }
});

// 请求拦截器
api.interceptors.request.use(
  config => {
    // 在发送请求之前做些什么
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  error => {
    // 对请求错误做些什么
    return Promise.reject(error);
  }
);

// 响应拦截器
api.interceptors.response.use(
  response => {
    // 对响应数据做点什么
    return response.data;
  },
  error => {
    // 对响应错误做点什么
    if (error.response.status === 401) {
      // 处理未授权错误
      router.push('/login');
    }
    return Promise.reject(error);
  }
);
```

### 1.2 请求方法
```js
// GET 请求
async function getUsers() {
  try {
    const response = await api.get('/users', {
      params: {
        page: 1,
        limit: 10
      }
    });
    return response.data;
  } catch (error) {
    console.error('获取用户列表失败:', error);
    throw error;
  }
}

// POST 请求
async function createUser(userData) {
  try {
    const response = await api.post('/users', userData);
    return response.data;
  } catch (error) {
    console.error('创建用户失败:', error);
    throw error;
  }
}

// PUT 请求
async function updateUser(id, userData) {
  try {
    const response = await api.put(`/users/${id}`, userData);
    return response.data;
  } catch (error) {
    console.error('更新用户失败:', error);
    throw error;
  }
}

// DELETE 请求
async function deleteUser(id) {
  try {
    await api.delete(`/users/${id}`);
  } catch (error) {
    console.error('删除用户失败:', error);
    throw error;
  }
}
```

## 2. 请求封装

### 2.1 统一请求服务
```js
// services/request.js
class RequestService {
  constructor(config) {
    this.api = axios.create(config);
    this.setupInterceptors();
  }

  setupInterceptors() {
    // 设置拦截器
  }

  async get(url, params = {}) {
    try {
      const response = await this.api.get(url, { params });
      return response.data;
    } catch (error) {
      this.handleError(error);
    }
  }

  async post(url, data = {}) {
    try {
      const response = await this.api.post(url, data);
      return response.data;
    } catch (error) {
      this.handleError(error);
    }
  }

  handleError(error) {
    // 统一错误处理
    const errorMessage = error.response?.data?.message || '请求失败';
    notification.error({
      message: '错误',
      description: errorMessage
    });
    throw error;
  }
}

export default new RequestService({
  baseURL: process.env.API_URL
});
```

### 2.2 API模块化
```js
// services/user.js
import request from './request';

export const userService = {
  getUsers(params) {
    return request.get('/users', params);
  },
  
  getUserById(id) {
    return request.get(`/users/${id}`);
  },
  
  createUser(data) {
    return request.post('/users', data);
  }
};

// services/auth.js
export const authService = {
  login(credentials) {
    return request.post('/auth/login', credentials);
  },
  
  logout() {
    return request.post('/auth/logout');
  }
};
```

## 3. 错误处理

### 3.1 全局错误处理
```js
// 错误处理中间件
function errorMiddleware() {
  return async (error) => {
    if (error.response) {
      // 服务器响应错误
      switch (error.response.status) {
        case 400:
          notification.error({ message: '请求参数错误' });
          break;
        case 401:
          notification.error({ message: '未授权，请登录' });
          router.push('/login');
          break;
        case 403:
          notification.error({ message: '拒绝访问' });
          break;
        case 404:
          notification.error({ message: '请求的资源不存在' });
          break;
        case 500:
          notification.error({ message: '服务器错误' });
          break;
        default:
          notification.error({ message: '未知错误' });
      }
    } else if (error.request) {
      // 请求发送失败
      notification.error({ message: '网络错误' });
    } else {
      // 请求配置错误
      notification.error({ message: '请求配置错误' });
    }
    return Promise.reject(error);
  };
}
```

### 3.2 重试机制
```js
// 请求重试配置
const retryConfig = {
  retry: 3,
  retryDelay: (retryCount) => {
    return retryCount * 1000; // 重试间隔递增
  },
  retryCondition: (error) => {
    // 只在网络错误时重试
    return axios.isAxiosError(error) && !error.response;
  }
};

// 添加重试拦截器
api.interceptors.response.use(null, async (error) => {
  const config = error.config;
  
  if (!config || !retryConfig.retry) {
    return Promise.reject(error);
  }
  
  config.__retryCount = config.__retryCount || 0;
  
  if (config.__retryCount >= retryConfig.retry) {
    return Promise.reject(error);
  }
  
  config.__retryCount += 1;
  
  const backoff = new Promise(resolve => {
    setTimeout(() => {
      resolve();
    }, retryConfig.retryDelay(config.__retryCount));
  });
  
  await backoff;
  return api(config);
});
```

## 4. 缓存策略

### 4.1 请求缓存
```js
// 简单的内存缓存
const cache = new Map();

function cacheRequest(key, promise, ttl = 60000) {
  if (!cache.has(key)) {
    const cacheData = {
      promise,
      timeout: setTimeout(() => {
        cache.delete(key);
      }, ttl)
    };
    cache.set(key, cacheData);
  }
  return cache.get(key).promise;
}

// 使用缓存
async function getCachedData(url, params) {
  const key = `${url}?${new URLSearchParams(params)}`;
  return cacheRequest(
    key,
    api.get(url, { params }),
    5 * 60 * 1000 // 5分钟缓存
  );
}
```

### 4.2 数据预加载
```js
// 预加载数据
function preloadData(urls) {
  return Promise.all(
    urls.map(url => 
      api.get(url)
        .then(response => cache.set(url, response))
        .catch(error => console.warn(`预加载失败: ${url}`, error))
    )
  );
}

// 在路由切换时预加载
router.beforeEach((to, from, next) => {
  if (to.meta.preload) {
    preloadData(to.meta.preload);
  }
  next();
});
```

## 5. 性能优化

### 5.1 请求合并
```js
// 请求队列
class RequestQueue {
  constructor() {
    this.queue = new Map();
    this.timeout = null;
  }

  add(key, request) {
    if (!this.queue.has(key)) {
      this.queue.set(key, []);
    }
    
    return new Promise((resolve, reject) => {
      this.queue.get(key).push({ resolve, reject });
      
      if (!this.timeout) {
        this.timeout = setTimeout(() => this.flush(), 50);
      }
    });
  }

  async flush() {
    const queue = new Map(this.queue);
    this.queue.clear();
    this.timeout = null;
    
    for (const [key, promises] of queue) {
      try {
        const response = await api.get(key);
        promises.forEach(({ resolve }) => resolve(response));
      } catch (error) {
        promises.forEach(({ reject }) => reject(error));
      }
    }
  }
}
```

### 5.2 并发控制
```js
// 并发限制
class ConcurrencyLimit {
  constructor(limit) {
    this.limit = limit;
    this.running = 0;
    this.queue = [];
  }

  async add(fn) {
    if (this.running >= this.limit) {
      await new Promise(resolve => this.queue.push(resolve));
    }
    
    this.running++;
    try {
      return await fn();
    } finally {
      this.running--;
      if (this.queue.length > 0) {
        this.queue.shift()();
      }
    }
  }
}

const limiter = new ConcurrencyLimit(3);

// 使用限制器
async function fetchWithLimit(url) {
  return limiter.add(() => api.get(url));
}
```

## 6. 文件上传

### 6.1 基本上传
```js
// 文件上传服务
const uploadService = {
  async uploadFile(file, onProgress) {
    const formData = new FormData();
    formData.append('file', file);
    
    return api.post('/upload', formData, {
      headers: {
        'Content-Type': 'multipart/form-data'
      },
      onUploadProgress: (progressEvent) => {
        const percentCompleted = Math.round(
          (progressEvent.loaded * 100) / progressEvent.total
        );
        onProgress?.(percentCompleted);
      }
    });
  }
};

// 使用上传服务
async function handleUpload(file) {
  try {
    const response = await uploadService.uploadFile(file, (progress) => {
      console.log(`上传进度: ${progress}%`);
    });
    return response.data.url;
  } catch (error) {
    console.error('上传失败:', error);
    throw error;
  }
}
```

### 6.2 分片上传
```js
// 分片上传服务
class ChunkUploadService {
  constructor(file, chunkSize = 1024 * 1024) {
    this.file = file;
    this.chunkSize = chunkSize;
    this.chunks = Math.ceil(file.size / chunkSize);
  }

  async upload() {
    const tasks = [];
    
    for (let i = 0; i < this.chunks; i++) {
      const chunk = this.file.slice(
        i * this.chunkSize,
        (i + 1) * this.chunkSize
      );
      tasks.push(this.uploadChunk(chunk, i));
    }
    
    const results = await Promise.all(tasks);
    return this.mergeChunks(results);
  }

  async uploadChunk(chunk, index) {
    const formData = new FormData();
    formData.append('chunk', chunk);
    formData.append('index', index);
    formData.append('filename', this.file.name);
    
    return api.post('/upload/chunk', formData);
  }

  async mergeChunks(chunks) {
    return api.post('/upload/merge', {
      filename: this.file.name,
      chunks: chunks.length
    });
  }
}
``` 